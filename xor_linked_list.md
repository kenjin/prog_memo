# XOR Linked List
###### tags: `XOR` `Linked List`

> An XOR linked list is a type of data structure used in computer programming. It takes advantage of the bitwise XOR operation to decrease storage requirements for doubly linked lists. - [Wiki](https://en.wikipedia.org/wiki/XOR_linked_list)


## Data Structure - XOR Linked List 
```c=
typedef struct _list
{
    int val;
    struct _list *link;
} list;
```
## XOR 特性

      addr:    0x01            0x02            0x04            0x08
      node:     n1              n2              n3              n4
      link:  null^0x02       0x01^0x04       0x02^0x08        0x04^null

#### 假設 linked list 為 "A - B - C" :
將 address 透過 **XOR** 達成 doubly linked list 之 `prev` & `next` 效果
- B 的 `link` 跟 address of C 進行 XOR 可得到 address of A (`B->prev = A`)
- B 的 `link` 跟 address of A 進行 XOR 可得到 address of C (`B->next = C`)

## Insertion to the head of linked list:
插入的重點在於調整 `link` 的值，若 linked list 插入第一個 node `AA`，則其 link 為 NULL。接著當 linked list 插入第二個 node `BB`，則必須更新原 `AA` link 值及 `BB` 的 link 值。(**正確的更新 link 值才能做後續的 XOR 進行 next/prev 切換**)

#### Insert first node [AA]

    addr:       0x01
	----------------
	 val:       "AA"
	link:       Null

#### Insert second node [BB]

	    0x02    0x01
	----------------------------
	    "BB"    "AA"
	    0x01    0x02
                  | 
                  +--> = XOR([BB], [AA].addr) = XOR(0x02, 0x00)


#### Insert third node [CC]

        0x04    0x02    0x01
	--------------------------------------------------------
	    "CC"    "BB"    "AA"
        0x02    0x05    0x02
	      |       |
          |       +---> XOR([CC], [BB].addr) = XOR(0x04, 0x01)
          |
	      +--> [BB]

針對 head(tail) node，可以發現 `link` 值為其後(前)一個 node 的 address，說到底就只是為了反應出 `XOR(addr1, addr1) = 0(or NULL)` 而已。

## C Implementation
```c=
#include <stdio.h>
#include <stdint.h>
#include <stdlib.h>

/*
 * #define XOR(a, b) (list *)((unsigned int)(a)^(unsigned int)(b))
 *
 * Don't do that. It causes warning: "cast from pointer to integer of
 * different size".
 *
 * Sol: include <stdint.h> and use intptr_t to handle casting
 *
 */
#define XOR(a, b) (list *)((intptr_t)(a)^(intptr_t)(b))

typedef struct _list
{
    int val;
    struct _list *link;
} list;

void insertHead(list **head, int data)
{
    list *newNode = malloc(sizeof(list));
    newNode->val = data;

    if (*head == NULL) {
        newNode->link = NULL;
    } else {
        /* Update original link of head node */
        (*head)->link = XOR((*head)->link, newNode);
        newNode->link = *head;
    }

    *head = newNode;
}

void deleteHead(list **head)
{
    if (!(*head))
        return;

    list *tmp = (*head)->link;
    /* Update the link of new head */
    if (tmp)
        tmp->link = XOR((*head), tmp->link);
    free(*head);
    *head = tmp;
}

void printXorList(list *head)
{
    if (!head)
        return;

    list *prev = NULL;
    while (head) {
        list *tmp = head;
        printf("%d ", head->val);
        head = XOR(prev, head->link);
        prev = tmp;
    }
    printf("\n");
}
```
<!-- Other important details discussed during the meeting can be entered here. -->
