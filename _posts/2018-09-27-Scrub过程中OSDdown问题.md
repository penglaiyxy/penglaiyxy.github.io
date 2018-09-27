---
layout:     post
title:      Scrub 导致 OSD DOWN问题分析
subtitle:   夏天的尾巴
date:       2018-09-27
author:     鱼鱼
header-img: img/post-bg-mma-0.jpg
catalog: true
tags:
    - ceph
    - osd
    - 漫谈
    - scrub
---


环境信息：

ceph 版本 0.94.5, centos 6.7

前阵子在测试环境发现一个OSD每次进行scrub，都会挂掉， core信息如下：
osd/osd_types.cc:4103： FAILED assert (clone_size.cout(clone))

ceph version 0.94.5:

 1. (SnapSet::get_clone_bytes(snapid_t))
 2. (ReplicatedPG::scrub(ScrubMap&, std::map, std::less, std::allocator >>> const &))
 3. (PG::scrub_compare_maps())
 4. (PG::chunky_scrub(ThreadPoll::TPHanle &))
 5. (PG::scrub(ThreadPool::TPHandle &))
 6. (OSD::ScrubWQ::process(PG *, ThreadPool::TPHanle &))
 7. (ThreadPool::worker(ThreadPool::WorkThread*))
 8. (ThreadPool::WorkThread::entry())
 9. /lib64/libpthread.so.0()
 10. (clone())

从代码上看发现在如下代码出现问题：

    uint64_t SnapSet::get_clone_bytes(snapid_t clone) const
    {
      assert(clone_size.count(clone));
      uint64_t size = clone_size.find(clone)->second;
  
      assert(clone_overlap.count(clone));
      const interval_set<uint64_t> &overlap = clone_overlap.find(clone)->second;
  
      for (interval_set<uint64_t>::const_iterator i = overlap.begin();
         i != overlap.end();
         ++i) {
        assert(size >= i.get_len());
        size -= i.get_len();
      }
      return size;
    }

大体意思是：

从Head对象中获取该object的Snapset信息，然后根据snap对象的snap_id查找该snap的clone信息，但是获取不到，ASSERT失败！

从网上找到类似的问题跟踪：

https://github.com/ceph/ceph/pull/7702

但是代码修改量太多，很多不是相关的，直接看环境，看下到底是哪里出问题了，
直接找到出问题的对象，发现该head对象的snapset为空，但是还存在一个snap对象；

使用如下命令查看snapset信息：

    attr -g ceph._ head对象 -q > /tmp/attr
    ceph-dencode type object_info_t export /tmp/attr decode dump_json

如果head对象的snapset为空，理论上不应该存在snap对象的；

从0.94.10版本看，如果出现类似问题，先检查是否期待有一个snap对象，如果没有期待，但是冒出一个snap对象，认为该snap对象是异常的，不会直接assert；

按照这个逻辑在0.94.5上修改后，OSD scrub运行正常了！

欢迎大家讨论！