# Rdb config

1. save \<seconds\> \<changes\>

    Trigger RDB save if `seconds` are passed and `changes` are made. 

2. rdbcompression yes

    Use LZF compression.
    
3. stop-writes-on-bgsave-error yes

    Stop writing if error. 
    
4. rdbchecksum yes

    Use CRC64 to check sum.
    
5. dbfilename dump.rdb

    RDB file name.
    
6. dir ./

    Directory of RDB file.

7. rdb-save-incremental-fsync yes

    Call `fsync` every 32MB is written.
