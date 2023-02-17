
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
- # VOS
  - ## layout
  - ## vos pool: 内存结构
    - vos_pool_create
    - 创建pmemobj实例（pmem，bmem）
    - 创建pool/cont 持久化B+tree实例(vos_pool_df, btr_root(pd_cont_root))
    - 初始化GC bin，初始化VEA
    - mlock，lock memory(scm size)
  - ## vos container
  - ## vos object
  - ## daos btree
  - ## evtree
    - ### R-tree
  - ## vos dtx
  - ## vos tree
  - ## ilog: Incarnation log wrappers for fetching the log and checking existence
  - ## vos timestamp: record timestamp table
- # BIO
  - ## SMD
- # VEA
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
