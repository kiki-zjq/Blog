# CMU 15213 C Lab

CMU 神课 CSAPP 的第一个 Lab，也是最简单的 Lab，主要目的是检测和锻炼学生对于 C 语言的掌握，毕竟后面的 Lab 大多数都要依靠 C 语言来写。

在这个 Lab 中我们使用类似于单向链表的结构创建了一个 Queue，这个 Queue 支持 FIFO 和 LIFO。我们总共需要完成 7 个 functions

- `queue_new`: Create a new, empty queue.
- `queue_free`: Free all storage used by a queue.
- `queue_insert_head`: Attempt to insert a new element at the head of the queue (LIFO discipline).
- `queue_insert_tail`: Attempt to insert a new element at the tail of the queue (FIFO discipline).
- `queue_remove_head`: Attempt to remove the element at the head of the queue.
- `queue_size`: Compute the number of elements in the queue.
- `queue_reverse`: Reorder the list so that the queue elements are reversed in order. This function should not allocate or free any list elements (either directly or via calls to other functions that allocate or free list elements.) Instead, it should rearrange the existing elements.

除了这 7 个 functions 以外，我们在完成的过程中会修改两个结构定义 `list_ele_t` 和 `queue_t`

我们可以先简单的看一下队列 `queue_t` 的结构定义

```c
typedef struct {
    /**
     * @brief Pointer to the first element in the queue, or NULL if the
     *        queue is empty.
     */
    list_ele_t *head;
    /*
     * TODO: You will need to add more fields to this structure
     *       to efficiently implement q_size and q_insert_tail
     */
    list_ele_t *tail;
    int size;
} queue_t;
```

`queue_t` 是一个指向整个队列的指针

初始代码里仅提供了 `head` 属性，用来指向 queue 的头节点

在这里，我们额外增加了 `tail` 属性和 `size` 属性，这是因为在后续完成跟 tail 相关的函数的时候，我们需要一个指向队尾的指针帮我们快速操作。同理，有一个 `queue_size` 函数要求我们返回队列中元素的数量，显然我们不希望每次调用都要从头到尾遍历整个 queue，这会造成 O(n) 的时间复杂度，因此我们专门增加一个 `size` 属性，将这一步的操作耗时降低到 O(1)

### Function 1. `queue_new`

```c
/**
 * @brief Allocates a new queue
 * @return The new queue, or NULL if memory allocation failed
 */
queue_t *queue_new(void) {
    queue_t *q = malloc(sizeof(queue_t));
    /* What if malloc returned NULL? -> We will return null directly */
    if (q) {
        q->head = NULL;
        q->tail = NULL;
        q->size = 0;
    }
    return q;
}
```

初始化一个新的队列，唯一需要注意的是当 malloc 函数返回 NULL 的时候，我们就不能再为指针分配属性了，而是需要直接返回 NULL

### Function 2. `queue_free`

```c
/**
 * @brief Frees all memory used by a queue
 * @param[in] q The queue to free
 */
void queue_free(queue_t *q) {
    /* How about freeing the list elements and the strings?*/
    /* Free queue structure */

    /* So we should traverse the queue from head to tail */
    if (!q) return;

    for (int i = 0; i < q->size; i++) {
        list_ele_t *cur = q->head;
        q->head = cur->next;
        free(cur->value);
        free(cur);
    }

    free(q);
}
```

这个函数要求我们清空 queue 所占用的所有内存，我们可以从头到尾遍历整条队列，然后在遍历的过程中，对每一个节点的相关属性调用 free 函数来进行内存的释放。

注意到每个节点的结构如下

```c
typedef struct list_ele {
    /**
     * @brief Pointer to a char array containing a string value.
     *
     * The memory for this string should be explicitly allocated and freed
     * whenever an element is inserted and removed from the queue.
     */
    char *value;

    /**
     * @brief Pointer to the next element in the linked list.
     */
    struct list_ele *next;
} list_ele_t;
```

每个节点不仅储存指向下一个节点的指针，也储存一个 value 值，对于这个 value，我们需要显示的通过调用 `malloc` / `free` 来分配 / 释放内存

### Function 3. `queue_insert_head`

```c
/**
 * @brief Attempts to insert an element at head of a queue
 *
 * This function explicitly allocates space to create a copy of `s`.
 * The inserted element points to a copy of `s`, instead of `s` itself.
 *
 * @param[in] q The queue to insert into
 * @param[in] s String to be copied and inserted into the queue
 *
 * @return true if insertion was successful
 * @return false if q is NULL, or memory allocation failed
 */
bool queue_insert_head(queue_t *q, const char *s) {
    if (!q) return false;
    list_ele_t *newh;
    /* What should you do if the q is NULL? */
    newh = malloc(sizeof(list_ele_t));
    /* Don't forget to allocate space for the string and copy it */
    /* What if either call to malloc returns NULL? */
    if (!newh) return false;

    char *news = malloc(sizeof(char) * (strlen(s) + 1));
    if (!news) {
        free(newh);
        return false;
    }

    strcpy(news, s);

    newh->value = news;

    newh->next = q->head;
    q->head = newh;

    if (!q->tail) q->tail = newh;
    ++q->size;

    return true;
}
```

这一个函数要求我们在队头插入一个节点。实际上分为了三个步骤

1. 调用 `malloc` 函数分配内存
    1. 注意不仅需要给节点指针分配内存，还需要给该节点包含的 string 分配内存
    2. 任何一步内存分配失败，我们都需要记得清空此前分配的所有内存，避免内存泄漏
2. 为新创建的 node 赋值
3. 操作指针，包括 node 的 next 指针，以及 queue 的头尾指针

### Function 4. `queue_insert_tail`

```c
/**
 * @brief Attempts to insert an element at tail of a queue
 *
 * This function explicitly allocates space to create a copy of `s`.
 * The inserted element points to a copy of `s`, instead of `s` itself.
 *
 * @param[in] q The queue to insert into
 * @param[in] s String to be copied and inserted into the queue
 *
 * @return true if insertion was successful
 * @return false if q is NULL, or memory allocation failed
 */
bool queue_insert_tail(queue_t *q, const char *s) {
    /* You need to write the complete code for this function */
    /* Remember: It should operate in O(1) time */
    if (!q) return false;
    list_ele_t *new_node = malloc(sizeof(list_ele_t));
    if (!new_node) return false;

    char *new_s = malloc(sizeof(char) * (strlen(s) + 1));
    if (!new_s) {
        free(new_node);
        return false;
    }

    strcpy(new_s, s);

    new_node->value = new_s;
    new_node->next = NULL;

    if (q->tail) {
        q->tail->next = new_node;
        q->tail = new_node;
    } else { // It is an empty queue
        q->head = new_node;
        q->tail = new_node;
    }

    ++q->size;
    return true;
}
```

和前一个函数几乎一样，只需要注意最后的操作变成了修改队列的尾指针即可

### Function 5. `queue_remove_head`

```c
/**
 * @brief Attempts to remove an element from head of a queue
 *
 * If removal succeeds, this function frees all memory used by the
 * removed list element and its string value before returning.
 *
 * If removal succeeds and `buf` is non-NULL, this function copies up to
 * `bufsize - 1` characters from the removed string into `buf`, and writes
 * a null terminator '\0' after the copied string.
 *
 * @param[in]  q       The queue to remove from
 * @param[out] buf     Output buffer to write a string value into
 * @param[in]  bufsize Size of the buffer `buf` points to
 *
 * @return true if removal succeeded
 * @return false if q is NULL or empty
 */
bool queue_remove_head(queue_t *q, char *buf, size_t bufsize) {
    /* You need to fix up this code. */
    if (!q || !q->head) {
        return false;
    }

    list_ele_t *cur = q->head, *nxt = cur->next;

    if (buf) { // buf is not NULL
        size_t i = 0;
        for (; i < bufsize - 1 && *(cur->value + i) != '\0'; i++) {
            *(buf + i) = *(cur->value + i);
        }
        *(buf + i) = '\0';
    }
    free(cur->value);
    free(cur);

    q->head = nxt;
    --q->size;

    return true;
}
```

在这个函数中，我们要释放当前队列的头节点，并且如果输入的 buf 不为 NULL 的话，我们需要将这个头节点的最多 bufsize - 1 个字符复制到 buf 中

### Function 6. `queue_size`

```c
/**
 * @brief Returns the number of elements in a queue
 *
 * This function runs in O(1) time.
 *
 * @param[in] q The queue to examine
 *
 * @return the number of elements in the queue, or
 *         0 if q is NULL or empty
 */
size_t queue_size(queue_t *q) {
    /* You need to write the code for this function */
    /* Remember: It should operate in O(1) time */
    if (!q || !q->head) return 0;
    return (size_t) q->size;
}
```

返回当前队列的 size，由于我们在 `queue_t` 结构中存储了这个属性，因此可以在 O(1) 时间内返回

### Function 7. `queue_reverse`

```c
/**
 * @brief Reverse the elements in a queue
 *
 * This function does not allocate or free any list elements, i.e. it does
 * not call malloc or free, including inside helper functions. Instead, it
 * rearranges the existing elements of the queue.
 *
 * @param[in] q The queue to reverse
 */
void queue_reverse(queue_t *q) {
    /* You need to write the code for this function */
    if (!q || !q->head) return;
    list_ele_t *old_head = q->head, *old_tail = q->tail;

    list_ele_t *cur = q->head, *pre = NULL, *nxt = NULL;
    while (cur) {
        nxt = cur->next;
        cur->next = pre;

        pre = cur;
        cur = nxt;
    }

    q->head = old_tail;
    q->tail = old_head;
}
```