
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
    - ### 数据结构
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
    - ### primary
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
    - ### secondary
- # Pool
- # Container
- # Obj
- # VOS
- # BIO
  - ## SMD
- # VEA
- # RDB
  - ## Raft
- # RSVC
- # Rebuild
