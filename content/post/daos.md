
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
  - ## primary group
  - ## secondary group
  - ## membership
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
