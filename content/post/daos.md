
- # CaRT
  - ## crt context
    - 数据结构
      ```c
      struct crt_context {
        d_list_t		 cc_link;	/** link to gdata.cg_ctx_list */
        int			 cc_idx;	/** context index */
        struct crt_hg_context	 cc_hg_ctx;	/** HG context */
        bool			 cc_primary;	/** primary provider flag */

        /* callbacks */
        void			*cc_rpc_cb_arg;
        crt_rpc_task_t		 cc_rpc_cb;	/** rpc callback */
        crt_rpc_task_t		 cc_iv_resp_cb;

        /** RPC tracking */
        /** in-flight endpoint tracking hash table */
        struct d_hash_table	 cc_epi_table;
        /** binheap for inflight RPC timeout tracking */
        struct d_binheap	 cc_bh_timeout;
        /** mutex to protect cc_epi_table and timeout binheap */
        pthread_mutex_t		 cc_mutex;

        /** timeout per-context */
        uint32_t		 cc_timeout_sec;
        /** HLC time of last received RPC */
        uint64_t		 cc_last_unpack_hlc;

        /** Per-context statistics (server-side only) */
        /** Total number of timed out requests, of type counter */
        struct d_tm_node_t	*cc_timedout;
        /** Total number of timed out URI lookup requests, of type counter */
        struct d_tm_node_t	*cc_timedout_uri;
        /** Total number of failed address resolution, of type counter */
        struct d_tm_node_t	*cc_failed_addr;

        /** Stores self uri for the current context */
        char			 cc_self_uri[CRT_ADDR_STR_MAX_LEN];
      };
      ```
  - ## crt group
    - 数据结构
    ```c
    // crt_gdata.cg_grp.gg_primary_grp
    struct crt_grp_priv {
      d_list_t		 gp_link; /* link to crt_grp_list */
      crt_group_t		 gp_pub; /* public grp handle */

      /* Link to a primary group; only set for secondary groups  */
      struct crt_grp_priv	*gp_priv_prim;

      /* List of secondary groups associated with this group */
      d_list_t		gp_sec_list;

      /*
       * member ranks, should be unique and sorted, each member is the rank
       * number within the primary group.
       */
      struct crt_grp_membs	gp_membs;
      /*
       * the version number of the group. Set my crt_group_version_set or
       * crt_group_mod APIs.
       */
      uint32_t		 gp_membs_ver;
      /*
       * this structure contains the circular list of member ranks.
       * It's used to store SWIM related information and should strictly
       * correspond to members in gp_membs.
       */
      struct crt_swim_membs	 gp_membs_swim;

      /* size (number of membs) of group */
      uint32_t		 gp_size;
      /*
       * logical self rank in this group, only valid for local group.
       * the gp_membs->rl_ranks[gp_self] is its rank number in primary group.
       * For primary group, gp_self == gp_membs->rl_ranks[gp_self].
       * If gp_self is CRT_NO_RANK, it usually means the group version is not
       * up to date.
       */
      d_rank_t		 gp_self;
      /* List of PSR ranks */
      d_rank_list_t		 *gp_psr_ranks;
      /* PSR rank in attached group */
      d_rank_t		 gp_psr_rank;
      /* PSR phy addr address in attached group */
      crt_phy_addr_t		 gp_psr_phy_addr;
      /* address lookup cache, only valid for primary group */
      struct d_hash_table	 *gp_lookup_cache;

      /* uri lookup cache, only valid for primary group */
      struct d_hash_table	 gp_uri_lookup_cache;

      /* Primary to secondary rank mapping table */
      struct d_hash_table	 gp_p2s_table;

      /* Secondary to primary rank mapping table */
      struct d_hash_table	 gp_s2p_table;

      /* set of variables only valid in primary service groups */
      uint32_t		 gp_primary:1, /* flag of primary group */
             gp_view:1, /* flag to indicate it is a view */
            /* Auto remove rank from secondary group */
             gp_auto_remove:1;

      /* group reference count */
      uint32_t		 gp_refcount;

      pthread_rwlock_t	 gp_rwlock; /* protect all fields above */
    };
    ```
    - ### primary: 集群内所有RANK的集合
      - 在crt初始化流程中调用crt_primary_grp_init创建gg_primary_grp
        - gp_membs(members)
          - cgm_list: 所有rank集合
          - cgm_linear_list: 所有有效rank链表集合
          - 更新流程
            - daos server等待daos engine ready 之后通过DRPC(MethodSetRank)调用engine执行crt_rank_self_set
              - gp_self: rank
              - grp_add_to_membs_list: 添加至cgm_list并更新cgm_linear_list
              - crt_grp_lc_uri_insert: 将所有crt context与该rank uri地址信息记入本地cache
            - crt_group_primary_modify: server Raft集群leader更新触发/join Timer触发/失败后会重试 (joinLoop:doGroupUpdate)
    - ### secondary: 一个Pool(ds pool)内的RANK集合
      - 创建(ds_mgmt_hdlr_tgt_create->pool_alloc_ref)/加载(ds_pool_start_all)Pool时创建secondary group
      - 更新流程
        - ds_pool_tgt_map_update，iv条件触发(poolmap变更)
          - update_pool_group(UP/UPIN/DRAIN)
        - 生成 Secondary <->Primary rank map 映射表
          - gp_p2s_table
          - gp_s2p_table
  - ## crt IV
    - map_distd: map分发ULT,等待唤醒s_map_dist_cv(ds_pool_extend_handler,ds_pool_update_handler,pool_svc_exclude_rank)
      - mgmt_svc_map_dist_cb
        - Primary Group内组播MGMT_TGT_MAP_UPDATE消息
      - pool_svc_map_dist_cb
        - rbd commit后触发ds_pool_iv_map_update
        - crt_ivsync_rpc_issue/crt_ivu_rpc_issue组播更新？
- # Placement
  - ## pool map
    - 数据结构
    ```c
    struct pool_map {
      /** protect the refcount */
      pthread_mutex_t		 po_lock;
      /** Current version of pool map */
      uint32_t		 po_version;
      /** refcount on the pool map */
      int			 po_ref;
      /** # domain layers */
      unsigned int		 po_domain_layers;
      /**
       * Sorters for the binary search of different domain types.
       * These sorters are in ascending order for binary search of sorters.
       */
      struct pool_comp_sorter	*po_domain_sorters;
      /** sorter for binary search of target */
      struct pool_comp_sorter	 po_target_sorter;
      /**
       * Tree root of all components.
       * NB: All components must be stored in contiguous buffer.
       */
      struct pool_domain	*po_tree;
      /**
       * number of currently failed pool components of each type
       * of component found in the pool
       */
      struct pool_fail_comp	*po_comp_fail_cnts;
      /* Current least in version from all UP/NEW targets. */
      uint32_t		po_in_ver;
      /* Current least fseq version from all DOWN targets. */
      uint32_t		po_fseq;
    };
    
    struct pl_jump_map {
      /** placement map interface */
      struct pl_map		jmp_map;
      /* Total size of domain type specified during map creation */
      unsigned int		jmp_domain_nr;
      /* # UPIN targets */
      unsigned int		jmp_target_nr;
      /* The dom that will contain no colocated shards */
      pool_comp_type_t	jmp_redundant_dom;
    };
    ```
    - fault domain
      - 命名: getDefaultFaultDomain->hostname: "/<hostname>" lowercase
      - createEngine获取配置或默认，调用systemJoin加入集群
    - component
    - state: UP, UP_IN, DOWN, DOWN_OUT, NEW, DRAIN, UNKNOWN
    - 创建/变更
      - ds_pool_create_handler->init_pool_metadata 触发 gen_pool_buf 产生初始化poolmap并持久化到rdb
        - PoolCreate
          - FaultDomainTree: 
          ```shell
                      ┌──────────┐
                      │  root    │
                   ┌──┴──────────┴───┐
                   │                 │
              ┌────▼───┐          ┌──▼─────┐
              │ node1  │          │ node2  │
              ├────────┴─┐        └─┬──────┴┐
              │          │          │       │
              │          │          │       │
           ┌──▼────┐ ┌───▼───┐ ┌────▼──┐  ┌─▼─────┐
           │rank0  │ │rank1  │ │rank2  │  │rank3  │
           └───────┘ └───────┘ └───────┘  └───────┘
          ```
          - compressed domain tree 元组 -> <level, ID, number of children>
          ```c
          struct d_fault_domain {
            uint32_t fd_level;	/** level in the fault domain tree >= 1 */
            uint32_t fd_id;		/** unique ID */
            uint32_t fd_children_nr; /** number of children */
          };
          // uint32_t array
          // |2,root,2||1,node1,2|1,node2,2|rand0|rank1|rank2|rank3|
          ```
          - pool map
            - map buffer(忽略root节点)
            ```c
            // layout on disk
            //|strct pool_component(domain)|struct pool_component(rank)|struct pool_component(target)|
            ```
      - ds_pool_tgt_map_update: poolmap 变更流程触发
  - ## placement map
    - ### jump map
      - 数据结构
      ```c
      struct pl_map {
        /** corresponding pool uuid */
        uuid_t			 pl_uuid;
        /** link chain on hash */
        d_list_t		 pl_link;
        /** protect refcount */
        pthread_spinlock_t	 pl_lock;
        /** refcount */
        int			 pl_ref;
        /** pool connections, protected by pl_rwlock */
        int			 pl_connects;
        /** type of placement map */
        pl_map_type_t		 pl_type;
        /** reference to pool map */
        struct pool_map		*pl_poolmap;
        /** placement map operations */
        struct pl_map_ops       *pl_ops;
      };
  
      struct pl_jump_map {
        /** placement map interface */
        struct pl_map		jmp_map;
        /* Total size of domain type specified during map creation */
        unsigned int		jmp_domain_nr;
        /* # UPIN targets */
        unsigned int		jmp_target_nr;
        /* The dom that will contain no colocated shards */
        pool_comp_type_t	jmp_redundant_dom;
      };
  
      struct pl_obj_shard {
        uint32_t	po_shard;	/* shard identifier */
        uint32_t	po_target;	/* target id */
        uint32_t	po_fseq;	/* The latest failure sequence */
        uint32_t	po_rebuilding:1, /* rebuilding status */
            po_reintegrating:1; /* reintegrating status */
      };
      struct pl_obj_layout {
        uint32_t		 ol_ver;
        uint32_t		 ol_grp_size;// target count per group
        uint32_t		 ol_grp_nr; // total group count
        uint32_t		 ol_nr;
        struct pl_obj_shard	*ol_shards;
      };
      ```
      - 创建
        - ds_pool_tgt_map_update 触发 pl_map_update
          - jump_map_create: 根据pool map创建新的placement map
            - redundant domain: 故障域默认为node
            - find all upin target, 查找当前poolmap中所有状态为UPIN的target
          - obj_layout_create: client端open对象时创建该对象对应的placement map layout
            - jump_map_obj_place: 决定该obj的数据分布
              - get_object_layout: 对每一个group选group size个target[<1,2,3>,<4,5,6>,<7,8,9>]
                - get_target: 依据shard递归选取一个target,crc(obj_key, shard_num/fail_num)避免重复
      - jumphash
      ```c
      uint32_t
      d_hash_jump(uint64_t key, uint32_t num_buckets)
      {
        int64_t z = -1;
        int64_t y = 0;

        while (y < num_buckets) {
          z = y;
          key = key * 2862933555777941757ULL + 1;
          y = (z + 1) * ((double)(1LL << 31) /
                   ((double)((key >> 33) + 1)));
        }
        return z;
      }
      ```
      - obj -> dk -> shard target: 依据DK从对象group layout选出冗余target
        - obj_update_shards_get: 
          - 副本: dk-> dk hash -> group idx: 用jump hash算法映射dk hash到对应group
          - EC: tgt_bitmap
        - obj_shards_2_fwtgts: 选出leader target(lower f_seq and healthy)
        
- # Pool
- # Container
- # Obj
  - ## layout
    ```shell  
       redun user.high(32)
          │    │
          ▼    ▼
       ┌─┬─┬──┬─┬────────┐
       │8│8│16│ │        │
       └─┴─┴──┴─┴────────┘
        ▲    ▲     ▲
        │    │     │
     otype nr_grps user.low(64)
    ```
  - generate oid 流程
    - oid_gen: 批量申请，且解决并发冲突
      - user.low: 用户低64位调用RPC到daos engine获取,每次获取一个low oid
        - ds_cont_oid_alloc_handler(CONT_OID_ALLOC)
          - oid_iv_reserve
            - oid_iv_ent_update: oid自增
      - user.high: 用户高32位由上层自增（类批量申请2^32）
      - daos_obj_generate_oid: 设置otype(object class), redun(冗余类型), nr_grps(group个数<target_nr/grp_size>)
- # VOS
  - ## layout
  - ## vos pool: 内存结构
    - vos_pool_create
    - 创建pmemobj实例（pmem，bmem）
    - 创建pool/cont 持久化B+tree实例(vos_pool_df, btr_root(pd_cont_root))
    - 初始化GC bin，初始化VEA
    - mlock，lock memory(scm size)
  - ## vos container
    - vos_cont_create: 在vos pool下创建container实例，以dbtree形式管理（持久化结构）
  - ## vos object
    - vos_obj_hold: 获取vos obj,先从lru cache查找，在从pmem查找，如果没有可以新建
  - ## daos btree
  - ## evtree
    - ### R-tree
  - ## vos dtx
  - ## vos tree
  - ## ilog: Incarnation log wrappers for fetching the log and checking existence
    - vos_ilog_update
    - vos_ilog_fetch
    - vos_ilog_punch
    - vos_ilog_check
    - cache
    - dkey
    - akey
  - ## vos timestamp: record timestamp table
- # BIO
  - ## SMD
    - sys_db
  - ## Blobstore
    - storage 抽象, blob/cluster/page/logicblock(512B/4K)
    ```shell
    +-----------------------------------------------------------------+
    |                              Blob                               |
    | +-----------------------------+ +-----------------------------+ |
    | |           Cluster           | |           Cluster           | |
    | | +----+ +----+ +----+ +----+ | | +----+ +----+ +----+ +----+ | |
    | | |Page| |Page| |Page| |Page| | | |Page| |Page| |Page| |Page| | |
    | | +----+ +----+ +----+ +----+ | | +----+ +----+ +----+ +----+ | |
    | +-----------------------------+ +-----------------------------+ |
    +-----------------------------------------------------------------+
    ```
    - media 格式, Supperblock: cluster 0 page 0 
    ```shell
    LBA 0                                   LBA N
    +-----------+-----------+-----+-----------+
    | Cluster 0 | Cluster 1 | ... | Cluster N |
    +-----------+-----------+-----+-----------+
    
    | Page 0 | Page 1 ... Page N |
    +--------+-------------------+
    | Super  |  Metadata Region  |
    | Block  |                   |
    +--------+-------------------+
    ```
  - init 初始化
    - bio_nvme_init: nvme_conf, shm_id, mem_size, hugepage_size, 服务进程初始化流程
      - spdk_bs_opts_init
        - cluster_sz: 1GB
        - num_md_pages: 20K, blobs per devices
        - max_channel_ops: 4K, max inflight blob IOs per io channel
      - bio_spdk_env_init
        - spdk_env_opts_init: options 设置
        - spdk_env_init: 注册dpdk驱动
        - spdk_unaffinitize_thread: remove 当前线程亲和性
        - spdk_thread_lib_init
          - spdk_mempool_create:  g_spdk_msg_mempool, spdk_msg
    - bio_xsctxt_alloc: bio context 首次初始化流程,Main XS
      - spdk_thread_create: 创建spdk thread抽象，bxc_thread -- spdk tls_thread
      - spdk_subsystem_init_from_json_config: 用nvme_conf配置加载所有BDEV
        - rpc_bdev_nvme_attach_controller
          - bdev_nvme_create(rpc_bdev_nvme_attach_controller_done)
            - spdk_nvme_connect_aync(connect_attach_cb)
              - nvme_driver_init
              - nvme_probe_ctx_init
              - nvme_probe_internal
                - connect_attach_cb
                  - nvme_ctrlr_get_by_name
                  - nvme_ctrlr_create
                    - nvme_ctrlr_create_done
                      - spdk_io_device_register(nvme_ctrlr)
                      - nvme_ctrlr_populate_namespaces: 枚举nvme ctrlr下所有ns，并创建BDEV
                        - nvme_ctrlr_populate_namespace
                          - nvme_ctrlr_populdate_standard_namespace
                            - nvme_bdev_create: bdev
                              - nvme_disk_create(ns): bdev->disk
                              - spdk_io_device_register(bdev)
                              - spdk_bdev_register(bdev->disk)
      - init_bio_bdevs
        - create_bio_bdev
          - spdk_bdev_open_ext: spdk_bdev_desc, bb_desc
          - load_blobstore
            - spdk_bdev_create_bs_dev_ext
              - spdk_bdev_open_ext
              - blob_bdev_init
                - read : bdev_blob_read
                - write : bdev_blob_write
            - spdk_bs_init/spdk_load
          - unload_blobstore
      - init_blobstore_ctxt
        - bio_init_health_monitoring: 盘健康监控服务
        - load_blobstore
        - spdk_thread_send_msg: transit BS state to SETUP
        - spdk_bs_alloc_io_channel: open IO channel for xstream
        - dma_buffer_create: 每个XS 32 chunks, 每个chunk 8M, 共2048个4K page
          - dma_alloc_chunk: bxc_dma_buf
            - spdk_dma_malloc
              - bio_chk_sz: 8M/4K, 2048
  - IO stack
    - nvme_rw: write
      - drain_inflight_ios
      - spdk_blob_io_write
        - bio_request_submit_op
          - blob_calculate_lba_and_lba_count
          - bs_batch_open
          - bs_batch_write_dev
            - channel->dev->write
              - bdev_blob_write
                - spdk_bdev_write_blocks
                  - bdev_channel_get_io
                  - bdev_io_init
                  - bdev_io_submit
                    - bdev_io_do_submit
                      - bdev->fn_table->submit_request: nvmelib_fn_table
                        - bdev_nvme_submit_request
                          - bdev_nvme_writev(qpair)
                            - nvme_qpair_submit_request
                              - nvme_pcie_qpair_submit_tracker
          - bs_batch_close
- # VEA: Version Block Allocator
  - metadata 只记录free extents到一个btree中（SCM），可以和index树一起用同一个事务更新，保证原子性
  - delayed atomicity（由元数据PMDK事务保证原子性，VEA记录DRAM&SCM两种空间分配元数据，无需记录Step1的undo日志）
    1. 在DRAM中预留更新数据的空间
    2. 通过RDMA传输client端数据到预留空间
    3. 将预留空间数据持久化并通过PMDK事务更新VOS index原数据
  - allocation hint
    - VEA假定一个可预测的IO负载模型： 顺序追加（per IO stream），调用者记录最后一次空间分配地址做为下次分配的入参，可解决空间碎片问题
  - 初始化
    - vea_format: 持久化数据结构
      - block size: 4K, first block for vea header
      - dbtree_create_inplace: create free extent tree, vsd_free_tree
        - insert initial free extent <vfe_blk_off(1), vea_free_extent(1, total, 0)>
        ```c
        struct vea_free_extent {
          uint64_t  vfe_blk_off;// block offset of the extent
          uint32_t  vfe_blk_cnt;// total blocks of the extent
          uint32_t  vfe_age;// monotonic timestamp
        }
        ```
      - dbtree_create_inplace: vsd_vec_tree, create allocated extent vector tree for no-contiguous allocation
    - vea_load
      - create_free_class: 创建大小类空间管理二叉堆/btree
        - d_binheap_create_inplace: vfc_heap，管理64M及以上的空间块
        - dbtree_create: vfc_size_btr， 管理64M以下空间块
      - dbtree_create: vsi_free_btr, create in-memory free extent tree
      - dbtree_create: vsi_vec_btr, create in-memory extent vector tree
      - dbtree_create: vsi_agg_btr, create in-memory aggregation tree
      - load_space_info
        - dbtree_open_inplace(vsd_free_tree, vsi_md_free_btr)
        - dbtree_open_inplace(vsd_vec_tree, vsi_md_vec_btr)
        - dbtree_iterate(vsi_md_free_btr, load_free_entry(cb))
          - verify_free_entry
          - compound_free: free extent to in-memory compound index
        - dbtree_iterate(vsi_md_vec_btr, load_vec_entry(cb))
          - verify_vec_entry
          - compound_vec_alloc: do nothing for now
  - 空间管理
    - vea_reserve
      - hint_get
      - reserve_hit: reserve from hit offset
        - dbtree_fetch(vsi_free_btr): 查询hint offset指向空间是否满足
        - compound_alloc
          - dbtree_delete: 可用空间恰好等于申请空间
          - free_class_remove/free_class_add：可用空间大于申请空间
      - reserve_single: reverve from the large extents(64M) or a small extent
        - reserve_small(vfc_size_btr): 查询>=申请空间的第一个记录（已排序）
        - 从大堆排序（vfc_heap）中取top entry来分配空间
      - reserve_vector：未实现，直接返NOSPACE错误
      - hint_update
  - vea 元数据持久化，内存将pmem修改记入TX，由umem_tx_end统一提交事务
    - vos_publish_blocks: vox_tx_end流程触发
      - process_resrvd_list
        - persistent_alloc: update pmem addr
        - compound_free: abort事务触发，释放预留的空间资源
        - hint_tx_publish: update hint allacation info
- # RDB
  - ## key space of a database
    ```c
    *   rdb_path_root_key {
    *       "containers" {
    *           5742bdea-90e2-4765-ad74-b7f19cb6d78f {
    *               "ghce"
    *               "ghpce"
    *               "lhes" {
    *                   5
    *                   12349875
    *               }
    *               "lres" {
    *                   0
    *                   10
    *               }
    *               "snapshots" {
    *               }
    *               "user.attr_a"
    *               "user.attr_b"
    *           }
    *       }
    *       "container_handles" {
    *           b0733249-0c9a-471b-86e8-027bcfccc6b1
    *           92ccc99c-c755-45f4-b4ee-78fd081e54ca
    *       }
    *   }
    ```
  - ## service
    - rdb_create
    - rdb_start
    - rdb_stop
    - Path: path 是一个keys组成的链表，绝对路径从rdb_path_root_key(代表root KVS)开始
      - rdb_path_init
      - rdb_path_clone
      - rdb_path_push: 将指定key插入到指定path的尾部
        - 字节流 stream buffer format: [<len1, key1, len1>] -> buffer: [<len1, key1, len1>, <len2, key2, len2>]
      - rdb_path_fini
  - ## transaction
    - 调用约束
    ```c
    * Caller locking rules:
     *
     *   rdb_tx_begin()
     *   rdlock(rl)
     *   rdb_tx_<query>()
     *   rdb_tx_<update>()
     *   wrlock(wl)		// must before commit(); may not cover update()s
     *   rdb_tx_commit()
     *   unlock(wl)		// must after commit()
     *   unlock(rl)		// must after all {rd,wr}lock()s; may before commit()
     *   rdb_tx_end()
  
    /* Update operation codes */
    enum rdb_tx_opc {
      RDB_TX_INVALID		= 0,
      RDB_TX_CREATE_ROOT	= 1,
      RDB_TX_DESTROY_ROOT	= 2,
      RDB_TX_CREATE		= 3,
      RDB_TX_DESTROY		= 4,
      RDB_TX_UPDATE		= 5,
      RDB_TX_DELETE		= 6,
      RDB_TX_LAST_OPC		= UINT8_MAX
    };
    struct rdb_tx_op {
      enum rdb_tx_opc		dto_opc;
      rdb_path_t		dto_kvs;
      d_iov_t			dto_key;
      d_iov_t			dto_value;
      struct rdb_kvs_attr    *dto_attr;
    };
    ```
  - rdb_tx_begin: 获取rdb实例句柄（ref+1），获取该rdb实例的tx实例（并发处理？）
  - rdb_tx_lookup: 从本地vos db中读取数据，用LRU cache增加查询效率
  - rdb_tx_update: 生成update op，调用rdb_tx_append以字节流形式挂到tx buffer尾部
  - rdb_tx_commit: 调用rdb_raft_append_apply，执行raft log replication
    - 发起端调用 rbd_raft_cb_send_appendentries RPC方式法送RDB_APPENDENTRIES消息
    - 接收端调用 raft_append_entries 同步日志
    - log_append 触发log_offer回调执行rdb_raft_log_offer更新tx事务日志到vos(rdb_tx_apply)
  - rdb_tx_end: 释放db实例，释放tx dt_entry内存空间
  - ## Raft
    - state
      - RAFT_STATE_LEADER
      - RAFT_STATE_FOLLOWER
      - RAFT_STATE_CANDIDATE
- # RSVC
  - 数据结构
  ```c
  /** List of all replicated service classes */
  enum ds_rsvc_class_id {
    DS_RSVC_CLASS_MGMT,
    DS_RSVC_CLASS_POOL,
    DS_RSVC_CLASS_TEST,
    DS_RSVC_CLASS_COUNT
  };
  ```
  - ds_mgmt_create_pool 控制面创建pool时调用ds_mgmt_pool_svc_create启动RSVC服务
    - select_svc_ranks: 选取副本数(rf*2+1) 冗余因子*2 + 1(未考虑故障域即target可能在同一个rank下)
    - ds_rsvc_dist_start: 广播RSVC_START到选取的副本target上，创建副本服务
      - ds_rsvc_start: ds_rsvc
        - rdb_create: 创建rdb服务实例并启动
          - vos_pool_create: 创建pool(uuid)对应的rdb vos pool, 无NVME存储
          - vos_cont_create: 创建pool(uuid)对应的rdb vos cont
          - rdb_raft_init: 创建raft log container
          - rdb_open_internal： 构建rdb实例，分配SCM空间（rdb池只写SCM）
      - mgmt/pool srv模块注册rscv服务实例到rsvc_hash表
- # Rebuild
