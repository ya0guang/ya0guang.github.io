---
layout: single
title: GPU在Confidential Computing中如何管理内存？
date: 2024-02-12 10:10:07
categories: "Tech"
tags:
- Research
- CS
- Linux Kernel
- TEE
header:
  teaser: 
---


## 从UVM分配开始

Path: `kernel-open/nvidia-uvm/uvm_va_space.c` 


注意这个：

```c
    if (uvm_conf_computing_mode_enabled(gpu)) {
        NvU32 gpu_index = uvm_id_gpu_index(gpu->id);
        status = uvm_conf_computing_dma_buffer_alloc(&gpu->conf_computing.dma_buffer_pool,
                                                     &va_space->gpu_unregister_dma_buffer[gpu_index],
                                                     NULL);
        if (status != NV_OK)
            goto done;

        gpu_can_access_sysmem = false;
    }
```

```c
// Note that the "VA space" in the function name refers to a UVM per-process VA space.
// (This is different from a per-GPU VA space.)
NV_STATUS uvm_va_space_register_gpu(uvm_va_space_t *va_space,
                                    const NvProcessorUuid *gpu_uuid,
                                    const uvm_rm_user_object_t *user_rm_device,
                                    NvBool *numa_enabled,
                                    NvS32 *numa_node_id)
{
    NV_STATUS status;
    uvm_va_range_t *va_range;
    uvm_gpu_t *gpu;
    uvm_gpu_t *other_gpu;
    bool gpu_can_access_sysmem = true;

    status = uvm_gpu_retain_by_uuid(gpu_uuid, user_rm_device, &gpu);
    if (status != NV_OK)
        return status;

    // Enabling access counters requires taking the ISR lock, so it is done
    // without holding the (deeper order) VA space lock. Enabling the counters
    // after dropping the VA space lock would create a window of time in which
    // another thread could see the GPU as registered, but access counters would
    // be disabled. Therefore, the counters are enabled before taking the VA
    // space lock.
    if (uvm_gpu_access_counters_required(gpu->parent)) {
        status = uvm_gpu_access_counters_enable(gpu, va_space);
        if (status != NV_OK) {
            uvm_gpu_release(gpu);
            return status;
        }
    }

    uvm_va_space_down_write(va_space);

    // Make sure the gpu hasn't been already registered in this va space
    if (uvm_processor_mask_test(&va_space->registered_gpus, gpu->id)) {
        status = NV_ERR_INVALID_DEVICE;
        goto done;
    }

    // Mixing coherent and non-coherent GPUs is not supported
    for_each_va_space_gpu(other_gpu, va_space) {
        if (uvm_parent_gpu_is_coherent(gpu->parent) != uvm_parent_gpu_is_coherent(other_gpu->parent)) {
            status = NV_ERR_INVALID_DEVICE;
            goto done;
        }
    }

    // The VA space's mm is being torn down, so don't allow more work
    if (va_space->disallow_new_registers) {
        status = NV_ERR_PAGE_TABLE_NOT_AVAIL;
        goto done;
    }

    if (uvm_conf_computing_mode_enabled(gpu)) {
        NvU32 gpu_index = uvm_id_gpu_index(gpu->id);
        status = uvm_conf_computing_dma_buffer_alloc(&gpu->conf_computing.dma_buffer_pool,
                                                     &va_space->gpu_unregister_dma_buffer[gpu_index],
                                                     NULL);
        if (status != NV_OK)
            goto done;

        gpu_can_access_sysmem = false;
    }

    uvm_processor_mask_set(&va_space->registered_gpus, gpu->id);
    va_space->registered_gpus_table[uvm_id_gpu_index(gpu->id)] = gpu;

    if (gpu->parent->isr.replayable_faults.handling) {
        uvm_processor_mask_set(&va_space->faultable_processors, gpu->id);
        // System-wide atomics are enabled by default
        uvm_processor_mask_set(&va_space->system_wide_atomics_enabled_processors, gpu->id);
    }

    // All GPUs have native atomics on their own memory
    processor_mask_array_set(va_space->has_native_atomics, gpu->id, gpu->id);

    // TODO: Bug 3252572: Support the new link type UVM_GPU_LINK_C2C
    if (gpu->parent->system_bus.link >= UVM_GPU_LINK_NVLINK_1) {
        processor_mask_array_set(va_space->has_nvlink, gpu->id, UVM_ID_CPU);
        processor_mask_array_set(va_space->has_nvlink, UVM_ID_CPU, gpu->id);
    }

    if (uvm_parent_gpu_is_coherent(gpu->parent)) {
        processor_mask_array_set(va_space->has_native_atomics, gpu->id, UVM_ID_CPU);

        if (gpu->mem_info.numa.enabled) {
            processor_mask_array_set(va_space->can_access, UVM_ID_CPU, gpu->id);
            processor_mask_array_set(va_space->accessible_from, gpu->id, UVM_ID_CPU);
            processor_mask_array_set(va_space->has_native_atomics, UVM_ID_CPU, gpu->id);
        }
    }

    // All processors have direct access to their own memory
    processor_mask_array_set(va_space->can_access, gpu->id, gpu->id);
    processor_mask_array_set(va_space->accessible_from, gpu->id, gpu->id);

    if (gpu_can_access_sysmem) {
        processor_mask_array_set(va_space->can_access, gpu->id, UVM_ID_CPU);
        processor_mask_array_set(va_space->accessible_from, UVM_ID_CPU, gpu->id);
    }

    processor_mask_array_set(va_space->can_copy_from, gpu->id, gpu->id);
    processor_mask_array_set(va_space->can_copy_from, gpu->id, UVM_ID_CPU);
    processor_mask_array_set(va_space->can_copy_from, UVM_ID_CPU, gpu->id);

    // Update the CPU/GPU affinity masks
    if (gpu->parent->closest_cpu_numa_node != -1) {
        uvm_gpu_id_t gpu_id;

        for_each_gpu_id(gpu_id) {
            uvm_cpu_gpu_affinity_t *affinity = &va_space->gpu_cpu_numa_affinity[uvm_id_gpu_index(gpu_id)];

            // If this is the first time this node is seen, take a new entry of
            // the array. Entries are never released in order to avoid having
            // to deal with holes.
            if (affinity->numa_node == -1) {
                UVM_ASSERT(uvm_processor_mask_empty(&affinity->gpus));
                affinity->numa_node = gpu->parent->closest_cpu_numa_node;
            }

            if (affinity->numa_node == gpu->parent->closest_cpu_numa_node) {
                uvm_processor_mask_set(&affinity->gpus, gpu->id);
                break;
            }
        }
    }

    status = register_gpu_nvlink_peers(va_space, gpu);
    if (status != NV_OK)
        goto cleanup;

    status = uvm_perf_heuristics_register_gpu(va_space, gpu);
    if (status != NV_OK)
        goto cleanup;

    uvm_for_each_va_range(va_range, va_space) {
        status = uvm_va_range_register_gpu(va_range, gpu);
        if (status != NV_OK)
            goto cleanup;
    }

    if (gpu->mem_info.numa.enabled) {
        *numa_enabled = NV_TRUE;
        *numa_node_id = (NvS32)uvm_gpu_numa_node(gpu);
    }
    else {
        *numa_enabled = NV_FALSE;
        *numa_node_id = -1;
    }

    goto done;

cleanup:
    // Clear out all of the processor mask bits. No VA ranges have mapped or
    // allocated anything on this GPU yet if we fail here, so we don't need
    // a deferred_free_list, mm, etc.
    unregister_gpu(va_space, gpu, NULL, NULL, NULL);

done:
    UVM_ASSERT(va_space_check_processors_masks(va_space));

    uvm_va_space_up_write(va_space);

    if (status != NV_OK) {
        // There is no risk of disabling access counters on a previously
        // registered GPU: the enablement step would have failed before even
        // discovering that the GPU is already registed.
        if (uvm_gpu_access_counters_required(gpu->parent))
            uvm_gpu_access_counters_disable(gpu, va_space);

        uvm_gpu_release(gpu);
    }

    return status;
}
```

## `uvm_conf_computing_dma_buffer_alloc`

Path: `kernel-open/nvidia-uvm/uvm_conf_computing.c`

```c
NV_STATUS uvm_conf_computing_dma_buffer_alloc(uvm_conf_computing_dma_buffer_pool_t *dma_buffer_pool,
                                              uvm_conf_computing_dma_buffer_t **dma_buffer_out,
                                              uvm_tracker_t *out_tracker)
{
    uvm_conf_computing_dma_buffer_t *dma_buffer = NULL;
    NV_STATUS status;

    UVM_ASSERT(dma_buffer_pool->num_dma_buffers > 0);

    // TODO: Bug 3385623: Heuristically expand DMA memory pool
    uvm_mutex_lock(&dma_buffer_pool->lock);
    if (list_empty(&dma_buffer_pool->free_dma_buffers)) {
        status = dma_buffer_pool_expand_locked(dma_buffer_pool);

        if (status != NV_OK) {
            uvm_mutex_unlock(&dma_buffer_pool->lock);
            return status;
        }
    }

    // We're guaranteed that at least one DMA stage buffer is available at this
    // point.
    dma_buffer = list_first_entry(&dma_buffer_pool->free_dma_buffers, uvm_conf_computing_dma_buffer_t, node);
    list_del_init(&dma_buffer->node);
    uvm_mutex_unlock(&dma_buffer_pool->lock);

    status = uvm_tracker_wait_for_other_gpus(&dma_buffer->tracker, dma_buffer->alloc->dma_owner);
    if (status != NV_OK)
        goto error;

    if (out_tracker)
        status = uvm_tracker_add_tracker_safe(out_tracker, &dma_buffer->tracker);
    else
        status = uvm_tracker_wait(&dma_buffer->tracker);

    if (status != NV_OK)
        goto error;

    uvm_page_mask_zero(&dma_buffer->encrypted_page_mask);
    *dma_buffer_out = dma_buffer;

    return status;

error:
    uvm_tracker_deinit(&dma_buffer->tracker);
    uvm_conf_computing_dma_buffer_free(dma_buffer_pool, dma_buffer, NULL);
    return status;
}
```

### `dma_buffer_pool_expand_locked`

Path: `kernel-open/nvidia-uvm/uvm_conf_computing.c`

```c
static NV_STATUS dma_buffer_pool_expand_locked(uvm_conf_computing_dma_buffer_pool_t *dma_buffer_pool)
{
    size_t i;
    uvm_gpu_t *gpu;
    size_t nb_to_alloc;
    NV_STATUS status = NV_OK;
    UVM_ASSERT(dma_buffer_pool->num_dma_buffers > 0);

    gpu = dma_buffer_pool_to_gpu(dma_buffer_pool);
    nb_to_alloc = dma_buffer_pool->num_dma_buffers;
    for (i = 0; i < nb_to_alloc; ++i) {
        uvm_conf_computing_dma_buffer_t *dma_buffer;

        status = dma_buffer_create(dma_buffer_pool, &dma_buffer);
        if (status != NV_OK)
            break;

        dma_buffer_pool_add(dma_buffer_pool, dma_buffer);
    }

    dma_buffer_pool->num_dma_buffers += i;

    if (i == 0)
        return status;

    return NV_OK;
}
```

### `dma_buffer_create`

Path: `kernel-open/nvidia-uvm/uvm_conf_computing.c`

```c
// Allocate and map a new DMA stage buffer to CPU and GPU (VA)
static NV_STATUS dma_buffer_create(uvm_conf_computing_dma_buffer_pool_t *dma_buffer_pool,
                                   uvm_conf_computing_dma_buffer_t **dma_buffer_out)
{
    uvm_gpu_t *dma_owner;
    uvm_conf_computing_dma_buffer_t *dma_buffer;
    uvm_mem_t *alloc = NULL;
    NV_STATUS status = NV_OK;
    size_t auth_tags_size = (UVM_CONF_COMPUTING_DMA_BUFFER_SIZE / PAGE_SIZE) * UVM_CONF_COMPUTING_AUTH_TAG_SIZE;

    dma_buffer = uvm_kvmalloc_zero(sizeof(*dma_buffer));
    if (!dma_buffer)
        return NV_ERR_NO_MEMORY;

    dma_owner = dma_buffer_pool_to_gpu(dma_buffer_pool);
    uvm_tracker_init(&dma_buffer->tracker);
    INIT_LIST_HEAD(&dma_buffer->node);

    status = uvm_mem_alloc_sysmem_dma_and_map_cpu_kernel(UVM_CONF_COMPUTING_DMA_BUFFER_SIZE, dma_owner, NULL, &alloc);
    if (status != NV_OK)
        goto err;

    dma_buffer->alloc = alloc;

    status = uvm_mem_map_gpu_kernel(alloc, dma_owner);
    if (status != NV_OK)
        goto err;

    status = uvm_mem_alloc_sysmem_dma_and_map_cpu_kernel(auth_tags_size, dma_owner, NULL, &alloc);
    if (status != NV_OK)
        goto err;

    dma_buffer->auth_tag = alloc;

    status = uvm_mem_map_gpu_kernel(alloc, dma_owner);
    if (status != NV_OK)
        goto err;

    *dma_buffer_out = dma_buffer;

    return status;

err:
    dma_buffer_destroy_locked(dma_buffer_pool, dma_buffer);
    return status;
}
```

## `uvm_mem_alloc_sysmem_dma_and_map_cpu_kernel`

Path: `kernel-open/nvidia-uvm/uvm_mem.h`

```c

// Helper for allocating sysmem DMA and mapping it on the CPU. This is useful
// for certain systems where the main system memory is encrypted
// (e.g., AMD SEV) and cannot be read from IO devices unless specially
// allocated using the DMA APIs.
//
// See uvm_mem_alloc()
static NV_STATUS uvm_mem_alloc_sysmem_dma_and_map_cpu_kernel(NvU64 size,
                                                             uvm_gpu_t *gpu,
                                                             struct mm_struct *mm,
                                                             uvm_mem_t **mem_out)
{
    NV_STATUS status;
    uvm_mem_t *mem;

    status = uvm_mem_alloc_sysmem_dma(size, gpu, mm, &mem);
    if (status != NV_OK)
        return status;

    status = uvm_mem_map_cpu_kernel(mem);
    if (status != NV_OK) {
        uvm_mem_free(mem);
        return status;
    }

    *mem_out = mem;
    return NV_OK;
}
```

### `uvm_mem_alloc_sysmem_dma`

Path: `kernel-open/nvidia-uvm/uvm_mem.h`

```c
// Helper for allocating sysmem in DMA zone using the default page size. The
// backing pages are not zeroed.
static NV_STATUS uvm_mem_alloc_sysmem_dma(NvU64 size, uvm_gpu_t *dma_owner, struct mm_struct *mm, uvm_mem_t **mem_out)
{
    uvm_mem_alloc_params_t params = { 0 };
    params.size = size;
    params.backing_gpu = NULL;
    params.dma_owner = dma_owner;
    params.page_size = UVM_PAGE_SIZE_DEFAULT;
    params.mm = mm;

    return uvm_mem_alloc(&params, mem_out);
}
```

#### `uvm_mem_alloc`

Path: `kernel-open/nvidia-uvm/uvm_mem.c`

```c
NV_STATUS uvm_mem_alloc(const uvm_mem_alloc_params_t *params, uvm_mem_t **mem_out)
{
    NV_STATUS status;
    NvU64 physical_size;
    uvm_mem_t *mem = NULL;
    bool is_unprotected = false;

    UVM_ASSERT(params->size > 0);

    mem = uvm_kvmalloc_zero(sizeof(*mem));
    if (mem == NULL)
        return NV_ERR_NO_MEMORY;

    mem->backing_gpu = params->backing_gpu;
    mem->dma_owner = params->dma_owner;
    UVM_ASSERT(!mem->dma_owner || !mem->backing_gpu);

    mem->size = params->size;
    mem->chunk_size = params->page_size;
    if (mem->chunk_size == UVM_PAGE_SIZE_DEFAULT)
        mem->chunk_size = mem_pick_chunk_size(mem);

    UVM_ASSERT(mem->chunk_size > 0);

    physical_size = UVM_ALIGN_UP(mem->size, mem->chunk_size);
    mem->chunks_count = physical_size / mem->chunk_size;

    if (params->is_unprotected)
        UVM_ASSERT(uvm_mem_is_vidmem(mem));

    is_unprotected = params->is_unprotected;

    status = mem_alloc_chunks(mem, params->mm, params->zero, is_unprotected);
    if (status != NV_OK)
        goto error;

    *mem_out = mem;
    return NV_OK;

error:
    uvm_mem_free(mem);
    return status;
}
```

#### `mem_alloc_chunks`

Path: `kernel-open/nvidia-uvm/uvm_mem.c`

```c
static NV_STATUS mem_alloc_chunks(uvm_mem_t *mem, struct mm_struct *mm, bool zero, bool is_unprotected)
{
    if (uvm_mem_is_sysmem(mem)) {
        gfp_t gfp_flags;
        uvm_memcg_context_t memcg_context;
        NV_STATUS status;

        UVM_ASSERT(PAGE_ALIGNED(mem->chunk_size));
        gfp_flags = sysmem_allocation_gfp_flags(get_order(mem->chunk_size), zero);
        if (UVM_CGROUP_ACCOUNTING_SUPPORTED() && mm)
            gfp_flags |= NV_UVM_GFP_FLAGS_ACCOUNT;

        uvm_memcg_context_start(&memcg_context, mm);
        if (uvm_mem_is_sysmem_dma(mem))
            status = mem_alloc_sysmem_dma_chunks(mem, gfp_flags);
        else
            status = mem_alloc_sysmem_chunks(mem, gfp_flags);

        uvm_memcg_context_end(&memcg_context);
        return status;
    }

    return mem_alloc_vidmem_chunks(mem, zero, is_unprotected);
}
```

#### `mem_alloc_sysmem_dma_chunks`

Path: `kernel-open/nvidia-uvm/uvm_mem.c`

```c
// This allocation is a non-protected memory allocation under Confidential
// Computing.
//
// There is a tighter coupling between allocation and mapping because of the
// allocator UVM must use. Hence, this function does the equivalent of
// uvm_mem_map_gpu_phys().
//
// In case of failure, the caller is required to handle cleanup by calling
// uvm_mem_free
static NV_STATUS mem_alloc_sysmem_dma_chunks(uvm_mem_t *mem, gfp_t gfp_flags)
{
    size_t i;
    NV_STATUS status;
    NvU64 *dma_addrs;

    UVM_ASSERT_MSG(mem->chunk_size == PAGE_SIZE,
                   "mem->chunk_size is 0x%x. PAGE_SIZE is only supported.",
                   mem->chunk_size);
    UVM_ASSERT(uvm_mem_is_sysmem_dma(mem));

    mem->sysmem.pages = uvm_kvmalloc_zero(sizeof(*mem->sysmem.pages) * mem->chunks_count);
    mem->sysmem.va = uvm_kvmalloc_zero(sizeof(*mem->sysmem.va) * mem->chunks_count);
    if (!mem->sysmem.pages || !mem->sysmem.va)
        goto err_no_mem;

    status = mem_alloc_dma_addrs(mem, mem->dma_owner);
    if (status != NV_OK)
        goto error;

    dma_addrs = mem->sysmem.dma_addrs[uvm_global_id_gpu_index(mem->dma_owner->global_id)];

    for (i = 0; i < mem->chunks_count; ++i) {
        mem->sysmem.va[i] = uvm_gpu_dma_alloc_page(mem->dma_owner->parent, gfp_flags, &dma_addrs[i]);
        if (!mem->sysmem.va[i])
            goto err_no_mem;

        mem->sysmem.pages[i] = uvm_virt_to_page(mem->sysmem.va[i]);
        if (!mem->sysmem.pages[i])
            goto err_no_mem;
    }

    sysmem_set_mapped_on_gpu_phys(mem, mem->dma_owner);

    return NV_OK;

err_no_mem:
    status = NV_ERR_NO_MEMORY;
error:
    mem_free_sysmem_dma_chunks(mem);
    return status;
}
```

#### `uvm_gpu_dma_alloc_page`

Path: `kernel-open/nvidia-uvm/uvm_gpu.c`

```c
void *uvm_gpu_dma_alloc_page(uvm_parent_gpu_t *parent_gpu, gfp_t gfp_flags, NvU64 *dma_address_out)
{
    NvU64 dma_addr;
    void *cpu_addr;

    cpu_addr = dma_alloc_coherent(&parent_gpu->pci_dev->dev, PAGE_SIZE, &dma_addr, gfp_flags);

    if (!cpu_addr)
        return cpu_addr;

    *dma_address_out = dma_addr_to_gpu_addr(parent_gpu, dma_addr);
    atomic64_add(PAGE_SIZE, &parent_gpu->mapped_cpu_pages_size);
    return cpu_addr;
}
```

Finally, `dma_alloc_coherent` calls the kernel.
But then how is the DMA memory allocated in an unprotected way?

### `uvm_mem_map_cpu_kernel`

Path: `kernel-open/nvidia-uvm/uvm_mem.c`

```c
NV_STATUS uvm_mem_map_cpu_kernel(uvm_mem_t *mem)
{
    NV_STATUS status;

    UVM_ASSERT(mem);
    UVM_ASSERT(mem_can_be_mapped_on_cpu_kernel(mem));

    if (uvm_mem_mapped_on_cpu_kernel(mem))
        return NV_OK;

    if (uvm_mem_is_sysmem(mem))
        status = mem_map_cpu_to_sysmem_kernel(mem);
    else
        status = mem_map_cpu_to_vidmem_kernel(mem);

    if (status != NV_OK)
        return status;

    mem_set_mapped_on_cpu_kernel(mem);

    return NV_OK;
}
```

#### `mem_map_cpu_to_sysmem_kernel`

Path: `kernel-open/nvidia-uvm/uvm_mem.c`

```c
    if (g_uvm_global.sev_enabled && uvm_mem_is_sysmem_dma(mem))
        prot = uvm_pgprot_decrypted(PAGE_KERNEL_NOENC);
```

```c
static NV_STATUS mem_map_cpu_to_sysmem_kernel(uvm_mem_t *mem)
{
    struct page **pages = mem->sysmem.pages;
    size_t num_pages = uvm_mem_physical_size(mem) / PAGE_SIZE;
    pgprot_t prot = PAGE_KERNEL;

    UVM_ASSERT(uvm_mem_is_sysmem(mem));

    // If chunk size is different than PAGE_SIZE then create a temporary array
    // of all the pages to map so that vmap() can be used.
    if (mem->chunk_size != PAGE_SIZE) {
        size_t page_index;
        pages = uvm_kvmalloc(sizeof(*pages) * num_pages);
        if (!pages)
            return NV_ERR_NO_MEMORY;
        for (page_index = 0; page_index < num_pages; ++page_index)
            pages[page_index] = mem_cpu_page(mem, page_index * PAGE_SIZE);
    }

    if (g_uvm_global.sev_enabled && uvm_mem_is_sysmem_dma(mem))
        prot = uvm_pgprot_decrypted(PAGE_KERNEL_NOENC);

    mem->kernel.cpu_addr = vmap(pages, num_pages, VM_MAP, prot);

    if (mem->chunk_size != PAGE_SIZE)
        uvm_kvfree(pages);

    if (!mem->kernel.cpu_addr)
        return NV_ERR_NO_MEMORY;

    return NV_OK;
}
```

#### `uvm_pgprot_decrypted`

Path: `kernel-open/nvidia-uvm/uvm_linux.h`

```c
// uvm_pgprot_decrypted is a GPL-aware version of pgprot_decrypted that returns
// the given input when UVM cannot use GPL symbols, or pgprot_decrypted is not
// defined. Otherwise, the function is equivalent to pgprot_decrypted. UVM only
// depends on pgprot_decrypted when the driver is allowed to use GPL symbols:
// both AMD's SEV and Intel's TDX are only supported in conjunction with OpenRM.
//
// It is safe to invoke uvm_pgprot_decrypted in KVM + AMD SEV-SNP guests, even
// if the call is not required, because pgprot_decrypted(PAGE_KERNEL_NOENC) ==
// PAGE_KERNEL_NOENC.
//
// pgprot_decrypted was added by commit 21729f81ce8a ("x86/mm: Provide general
// kernel support for memory encryption") in v4.14 (2017-07-18)
static inline pgprot_t uvm_pgprot_decrypted(pgprot_t prot)
{
#if defined(pgprot_decrypted)
        return pgprot_decrypted(prot);
#endif

   return prot;
}
```

The memory is decrypted by the kernel at here: `pgprot_decrypted`.


