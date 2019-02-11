# Dictht

## Structure

|Field |Type | Purpose |
| --- | --- | --- |
| table | dictEntry** | slots, array of (entry linked list)|
| size | unsigned long | length of slots |
| sizemask | unsigned long | to calculate which slot index an entry is in |
| used | unsigned long | number of elements |