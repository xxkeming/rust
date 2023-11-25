## 源码下载
```
git clone https://github.com/Rust-for-Linux/linux -b rust-dev --depth 1
```

## 项目目标
```
rust实现基于内存的块设备模块
```

## blk-mq的binding添加,部分封装
### rust实现基于内存的块设备模块
### bindings_helper.h 添加需要引用的头文件
```
#include <linux/blkdev.h>
#include <linux/blk-mq.h>
```
### rust/kernel目录添加blkmq.rs,并重新编译内核
```
use crate::bindings;
use crate::sync::LockClassKey;
use alloc::boxed::Box;

/// gendisk
pub struct GenDisk(*mut bindings::gendisk);

unsafe impl Sync for GenDisk {}
unsafe impl Send for GenDisk {}

impl core::ops::Deref for GenDisk {
    type Target = bindings::gendisk;

    fn deref(&self) -> &Self::Target {
        unsafe { &*self.0 }
    }
}

impl core::ops::DerefMut for GenDisk {

    fn deref_mut(&mut self) -> &mut Self::Target {
        unsafe { &mut *self.0 }
    }
}

impl GenDisk {

    pub fn set_name(&mut self, name: &[u8]) {
        
        unsafe {
            core::ptr::copy_nonoverlapping(name.as_ptr(), self.disk_name.as_mut_ptr() as *mut u8, name.len());
        }
    }

    pub fn set_capacity(&mut self, sector: u64) {
        
        unsafe {
            bindings::set_capacity(&mut **self, sector)
        }
    }

    pub fn add(&mut self) -> isize {

        unsafe {
            bindings::device_add_disk(core::ptr::null_mut(), &mut **self, core::ptr::null_mut()) as isize
        }
    }
}

impl Drop for GenDisk {

    fn drop(&mut self) {
        unsafe {
            bindings::put_disk(&mut **self) 
        }
    }
}

/// blk mq tag set
pub struct Tagset(Box<bindings::blk_mq_tag_set>);

unsafe impl Sync for Tagset {}
unsafe impl Send for Tagset {}

impl Default for Tagset {

    fn default() -> Self {
        Self ( Box::try_new(bindings::blk_mq_tag_set::default()).unwrap() )
    }
}

impl Tagset {

    pub fn alloc(&mut self) -> isize {

        unsafe { bindings::blk_mq_alloc_tag_set(&mut **self) as isize }
    }

    pub fn alloc_disk(&mut self) -> Option<GenDisk> {

        static KEY: LockClassKey = LockClassKey::new();	
        
        unsafe 
        {
            let disk = crate::error::from_err_ptr::<bindings::gendisk>(bindings::__blk_mq_alloc_disk(&mut **self, core::ptr::null_mut(), KEY.as_ptr()));
            if let Ok(disk) = disk {
                Some(GenDisk(disk))
            } else {
                None
            }
        }
    }
}

impl Drop for Tagset {

    fn drop(&mut self) {

        unsafe {
            /*if self.driver_data != core::ptr::null() {
                bindings::vfree(self.driver_data);
            }*/
            bindings::blk_mq_free_tag_set(&mut **self)
        }
    }
}

impl core::ops::Deref for Tagset {
    type Target = bindings::blk_mq_tag_set;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

impl core::ops::DerefMut for Tagset {

    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.0
    }
}

/// blk mq ops
pub struct Ops(Box<bindings::blk_mq_ops>);

unsafe impl Sync for Ops {}
unsafe impl Send for Ops {}

impl Ops {

    pub fn new() -> Self {
        Self ( Box::try_new(bindings::blk_mq_ops::default()).unwrap() )
    }
}

impl core::ops::Deref for Ops {
    type Target = bindings::blk_mq_ops;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

impl core::ops::DerefMut for Ops {

    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.0
    }
}


/// blk mq ops
pub struct BdOps(Box<bindings::block_device_operations>);

unsafe impl Sync for BdOps {}
unsafe impl Send for BdOps {}

impl BdOps {

    pub fn new(module: &mut bindings::module ) -> Self {

        let mut ops = Box::try_new(bindings::block_device_operations::default()).unwrap();
        ops.owner = module;
        Self ( ops )
    }
}

impl core::ops::Deref for BdOps {
    type Target = bindings::block_device_operations;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

impl core::ops::DerefMut for BdOps {

    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.0
    }
}
```

### 以动态的方式构造ramdisk模块
### Makefile文件
```
ramdisk-objs := vlk_queue.o rustblk.o
obj-m := ramdisk.o
#obj-m := blk.o

PWD := $(shell pwd)
ARCH = x86_64
#KDIR = /lib/modules/$(shell uname -r)/build
KDIR = /home/parallels/linux/build

default:
	$(MAKE) ARCH=$(ARCH) LLVM=1 -C $(KDIR) M=$(PWD)/ modules
clean:
	$(MAKE) ARCH=$(ARCH) LLVM=1 -C $(KDIR) M=$(PWD)/ clean
```

### vlk_queue.c 部分函数太复杂,还是c实现方便
```
#include <linux/module.h>
#include <linux/blkdev.h>
#include <linux/blk-mq.h>

blk_status_t vblk_queue_rq(struct blk_mq_hw_ctx *hctx, const struct blk_mq_queue_data *bd)
{
    struct request *rq = bd->rq;
    struct bio_vec bvec;
    struct req_iterator iter;
    u64 offset;
    u32 len;
    u8 *buf;

    //pr_info("vblk_queue_rq\n");

    blk_mq_start_request(rq);

    offset = blk_rq_pos(rq) * 512;

    rq_for_each_segment(bvec, rq, iter) {
        len = bvec.bv_len;
        buf = page_address(bvec.bv_page) + bvec.bv_offset;
        if (rq_data_dir(rq) == READ) {
            memcpy(buf, hctx->driver_data + offset, len);
        } else {
            memcpy(hctx->driver_data + offset, buf, len);
        }
        offset += len;
    }

    blk_mq_complete_request(rq);

    return BLK_STS_OK;
}
```

### rustblk.rs
```
#![allow(missing_docs)]
#![allow(improper_ctypes)]

use kernel::prelude::*;
use kernel::blkmq;
use kernel::bindings;
use kernel::error;

module! {
    type: RustBlock,
    name: "rust_block",
    author: "keming",
    description: "block module in rust",
    license: "GPL",
}

unsafe extern "C" fn vblk_open(_disk: *mut bindings::gendisk, _mode: bindings::blk_mode_t) -> i32
{
    pr_info!("vblk_open\n");
    return 0;
}

unsafe extern "C" fn vblk_release(_disk: *mut bindings::gendisk)
{
    pr_info!("vblk_release\n");
}


unsafe extern "C" fn vblk_init(hctx: *mut bindings::blk_mq_hw_ctx, data: *mut core::ffi::c_void, _arg3: u32) -> i32 {

    unsafe {
        pr_info!("vblk_init {:#x}\n", data as usize);
        (*hctx).driver_data = data;
    }
    
    0
}

//* 
extern "C" {
    fn vblk_queue_rq(hctx: *mut bindings::blk_mq_hw_ctx, bd: *const bindings::blk_mq_queue_data) -> bindings::blk_status_t;
}
//*/

/* 
unsafe extern "C" fn vblk_queue_rq(hctx: *mut bindings::blk_mq_hw_ctx, bd: *const bindings::blk_mq_queue_data) -> bindings::blk_status_t {

    unsafe {
        pr_info!("queue_rq data: {:#x}\n", (*hctx).driver_data as usize);
        
        let rq = (*bd).rq;
        let mut iter = bindings::req_iterator::default();

        bindings::blk_mq_start_request(rq);
        let offset = (*rq).__sector * 512;

        iter.bio = (*rq).bio;

        loop {
            if iter.bio == core::ptr::null_mut() {
                break;
            }

            iter.iter = (*iter.bio).bi_iter;
            loop {
                if iter.iter.bi_size < 1 {
                    break;
                }

                let mut bv = bindings::bio_vec::default();
                bv.bv_page = 
                iter.iter.bio.bi_io_vec
                iter.iter

                #define bvec_iter_bvec(bvec, iter)				\
                ((struct bio_vec) {						\
                    .bv_page	= bvec_iter_page((bvec), (iter)),	\
                    .bv_len		= bvec_iter_len((bvec), (iter)),	\
                    .bv_offset	= bvec_iter_offset((bvec), (iter)),	\
                })

            }

            iter.bio = (*iter.bio).bi_next;
        }
        
        bindings::blk_mq_complete_request(rq);
    }
    bindings::BLK_STS_OK as bindings::blk_status_t
}
*/

unsafe extern "C" fn vblk_complete(rq: *mut bindings::request)
{
    unsafe {
        //pr_info!("complete {:#x} {:#x}", rq as usize, core::mem::size_of::<bindings::request>());
        //pr_info!("complete {:#x}", rq.add(1) as usize);

        let data = *(rq.add(1) as *mut usize).add(1);
        //pr_info!("complete kfree: {:#x}", data);

        if data != 0 {
            bindings::kfree(data as *const _);
        }

        bindings::blk_mq_end_request(rq, bindings::BLK_STS_OK as u8)
    }
}

struct RustBlock {
    ops: blkmq::Ops,
    bdops: blkmq::BdOps,
    tagset: blkmq::Tagset,
    disk: blkmq::GenDisk
}

impl kernel::Module for RustBlock {

    fn init(_module: &'static ThisModule) -> Result<Self> {

        let mut ops = blkmq::Ops::new();
        ops.init_hctx = Some(vblk_init);
        ops.queue_rq = Some(vblk_queue_rq);
        ops.complete = Some(vblk_complete);

        let mut tagset = blkmq::Tagset::default();
        tagset.ops = &*ops;
        tagset.nr_hw_queues = 4;
        tagset.queue_depth = 64;
        tagset.numa_node = bindings::NUMA_NO_NODE;
        tagset.cmd_size = 0;
        tagset.flags = bindings::BLK_MQ_F_SHOULD_MERGE;
        tagset.driver_data = unsafe { bindings::vzalloc(1024 * 1024 * 32) };

        if tagset.alloc() != 0 {
            unsafe { bindings::vfree(tagset.driver_data) };

            pr_err!("Failed to allocate tag set\n");
            return Err(error::code::EPERM);
        }

        let mut disk = match tagset.alloc_disk() {
            Some(r) => r,
            _ => {
                unsafe { bindings::vfree(tagset.driver_data) };

                pr_err!("Failed to add disk\n");
                return Err(error::code::EPERM);
            }
        };
        let mut bdops = unsafe { blkmq::BdOps::new(&mut bindings::__this_module) };
        bdops.open = Some(vblk_open);
        bdops.release = Some(vblk_release);

        disk.major = 0;
        disk.first_minor = 0;
        disk.fops = &*bdops;
        disk.set_name(b"vblk");
        disk.set_capacity((1024 * 1024 * 32) / 512);
        
        if disk.add() != 0 {

            unsafe { bindings::vfree(tagset.driver_data) };

            pr_err!("Failed to add disk\n");
            return Err(error::code::EPERM);
        }

        pr_info!("Rust module init {:p}\n", &*ops);

        Ok(RustBlock {ops, bdops, tagset, disk})
    }
}

impl Drop for RustBlock {
    
    fn drop(&mut self) {
        
        unsafe { bindings::vfree(self.tagset.driver_data) };

        pr_info!("Rust module drop {} {:#x} {:p} {:p} {:p}\n", self.tagset.nr_maps, self.tagset.ops as usize, &*self.ops, &*self.bdops, &self.disk);
    }
}
```

### 编译,打包到busybox,加载效果结果
<img width="1061" alt="image" src="https://github.com/xxkeming/rust/assets/11630632/11b51141-4ddb-431f-b0f5-de6569d1cfa9">

