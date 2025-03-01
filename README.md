## UID: 006160501

## HASH HASH HASH

This lab we are using two different mutual exclusion locking techniques on two different hash tables that are implemented using singly linked lists. Both time performances are to be compared during base hash table.
## Building

Type make in shell to make the: hash-table-common.o hash-table-base.o hash-table-v1.o hash-table-v2.o hash-table-tester.o hash-table-tester 
The only two files that have been changed apart from the README should be hash-table-v1.c/ hash-table-v2.c

## Running
Run 
```shell
 ./hash_table_tester -t _thread-count_ -s _element-count_

example: ./hash-table-tester -t 4 -s 99999
```

You should get some output like this with different values:

```shell
Generation: 94,073 usec
Hash table base: 1,903,711 usec
  - 0 missing
Hash table v1: 2,645,445 usec
  - 0 missing
Hash table v2: 421,611 usec
  - 0 missing
```

## V1
This version focuses on correctness and using one mutex lock. I added this single mutex in hash_table_v1_create(), as my lock intends to protect the whole hash table rather than the individual buckets or list item entries. As you can see in the example below, the v1 table is significantly slower than the base table. This is because we are using a single lock on the whole table. Meaning even though we have different buckets and one thread will be only working on one bucket at a time, all the other threads have to wait until the locked thread finishes that bucket and unlocks the whole table. For example, I have list_head number 1 changing by thread 1, list_head number 1 changing by thread 2, and list_head number 3 changing by thread 3. If thread 1 acquires the lock, thread 2 and 3 has to wait, even though thread 3 would be modyfying a different list_head's list and not affecting thread 1. The only benefit is that thread 1 will have no interuption from thread 2/3 eliminating the race condition (correctness) but lacking on performance (which will be addressed in V2)

```shell
./hash-table-tester -t 4 -s 99999
Generation: 81,172 usec
Hash table base: 1,388,338 usec
  - 0 missing
Hash table v1: 2,481,845 usec
  - 0 missing
Hash table v2: 438,060 usec
  - 0 missing
  ```
## V2
Here I added the lock in hash_table_entry. The reason why I didn't do it in list_entry itself because the chances of two thread race condition on the key value pair itself is less than the chances of a race condition on a bucket. If I had done a lock on list_entry, we may still have a race condition where two threads modify the head itself, and adding locks to all -s <element_count> of elements will create too much of a overhead. 

In the hash_table_v2_add_entry I initially had the lock set before the line	"struct list_head *list_head = &hash_table_entry->list_head;" where a thread aquires a list_head. Then I realized that even if a thread aquires a list_head, the actual address of the list_head doesn't change, and the thread would follow the list_head pointer inside the lock, so there won't be any race condition on the address list_head itself. This is why the lock is now AFTER the code line above. This significantly increased my number of chances of V2 being (-t x -1 ) times faster.

The way I checked to make sure my understanding was correct was adding:

```shell
printf("Before insertion, list_head: %p\n", (void*)list_head);
SLIST_INSERT_HEAD(list_head, list_entry, pointers);
printf("After insertion, list_head: %p\n", (void*)list_head);
```

If the list_head address value itself wasn't changed, then we know that the other thread can still access list_head because it has the pointer of list_head, not the old value list_head pointed to. That value would be accessed within the lock. 
--------------------------------------------------------------------------------------------------------------
Another change I made was with this code below. I realized that both conditions (if you found a key you change the value, but if there is no key you set a list entry) need a unlock. Previously I only used lock for the later condition which caused correctness issues.

This is because if the thread finds that there is a key, it should update and return. But returning without unlocking causes other threads unable to access that list. For example, lets say thread 1 finds a key called "apple" in a list_head address "12345". If thread 1 changes apple's value but returns without unlocking, there is now a barrier for other threads to go into "12345", which make other threads stop as they are waiting for the barrier to unlock. So instead of putting two pieces of the same code, I created a if, else statement, removing the previous return condition and having both if else case fall into the unlocking code. 

```shell
if (list_entry != NULL) {
		list_entry->value = value;
	} else {
	list_entry = calloc(1, sizeof(struct list_entry));
	list_entry->key = key;
	list_entry->value = value;
	SLIST_INSERT_HEAD(list_head, list_entry, pointers);

	 error = pthread_mutex_unlock(&hash_table_entry->lock); // unclock for both cases updating or inserting value
		if (error!= 0){
			exit(error);
		} 
	}

```
-------------------------------------------------------------------------------------------------------------
Now, I had to destroy the locks inside the for loop for (size_t i = 0; i < HASH_TABLE_CAPACITY; ++i) as each rotation of the loop accesses each entry, and we need to access each entry[i] (list) of the loop so we can destroy the lock that was associated with it. This differs with previous implementation of V1 where since we had 1 lock, we only need to demolish the lock once. I also checked if this is correct using valgrind.
-----------------------------------------------------------------------------------------------------------------
All of these changes are the reason why V2 works better than Base and V1. 

You can see with the code below, V2 <= Base/(-t x-1). Here 2,120,459 (v2) is less than 2,498,971.33 (base/3) which means our performance is correct.
```shell
sh-5.1$ ./hash-table-tester -t 4 -s 199999
Generation: 161,676 usec
Hash table base: 7,496,914 usec
  - 0 missing
Hash table v1: 11,088,402 usec
  - 0 missing
Hash table v2: 2,120,459 usec
  - 0 missing
```


## Cleaning up

Run make clean to remove the executables
```shell
make clean
```
