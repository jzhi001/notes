# RDB



## call chain

* rioWrite
* rdbWriteRaw
* rdbSaveType | rdbSaveMillisecondTime | rdbSaveLen | rdbSaveLzfBlob
* rdbSaveRawString | rdbLoadLzfStringObject 
| rdbSaveLongLongAsStringObject | rdbSaveStringObject
| rdbSaveDoubleValue | rdbSaveBinaryDoubleValue | rdbSaveObjectType

* Stream: rdbSaveStreamPEL | rdbSaveStreamConsumers
* Robj: rdbSaveObject
* Redis String: rdbSaveKeyValuePair
* rdbSaveAuxField

* Entrance: rdbSave | rdbSaveBackground -> rdbSaveRio

## rdbSaveInfo

```C
typedef struct rdbSaveInfo {
    /* Used saving and loading. */
    int repl_stream_db;  /* DB to select in server.master client. */

    /* Used only loading. */
    int repl_id_is_set;  /* True if repl_id field is set. */
    char repl_id[CONFIG_RUN_ID_SIZE+1];     /* Replication ID. */
    long long repl_offset;                  /* Replication offset. */
} rdbSaveInfo;

#define RDB_SAVE_INFO_INIT {-1,0,"000000000000000000000000000000",-1}
```

### rio::io

```C
struct {
    FILE *fp;
    off_t buffered; /* Bytes written since last fsync. */
    off_t autosync; /* fsync after 'autosync' bytes written. */
} file;
```

### fields in Server

```C
/* RDB / AOF loading information */
int loading;                /* We are loading data from disk if true */
off_t loading_total_bytes;
off_t loading_loaded_bytes;
time_t loading_start_time;
off_t loading_process_events_interval_bytes;

 /* RDB persistence */
long long dirty;                /* Changes to DB from the last save */
long long dirty_before_bgsave;  /* Used to restore dirty on failed BGSAVE */
pid_t rdb_child_pid;            /* PID of RDB saving child */
struct saveparam *saveparams;   /* Save points array for RDB */
int saveparamslen;              /* Number of saving points */
char *rdb_filename;             /* Name of RDB file */
int rdb_compression;            /* Use compression in RDB? */
int rdb_checksum;               /* Use RDB checksum? */
time_t lastsave;                /* Unix time of last successful save */
time_t lastbgsave_try;          /* Unix time of last attempted bgsave */
time_t rdb_save_time_last;      /* Time used by last RDB save run. */
time_t rdb_save_time_start;     /* Current RDB save start time. */
int rdb_bgsave_scheduled;       /* BGSAVE when possible if true. */
int rdb_child_type;             /* Type of save by active child. */
int lastbgsave_status;          /* C_OK or C_ERR */
int stop_writes_on_bgsave_err;  /* Don't allow writes if can't BGSAVE */
int rdb_pipe_write_result_to_parent; /* RDB pipes used to return the state */
int rdb_pipe_read_result_from_child; /* of each slave in diskless SYNC. */
```

