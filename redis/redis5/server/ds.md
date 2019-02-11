# Server DS and Fields

## RDB 

```C
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


struct saveparam {
    time_t seconds;
    int changes;
};

```

* **server.dirty**    Number of changes since last persistence.

 