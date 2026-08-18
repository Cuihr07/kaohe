# kaohe
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

#define MEM_POOL_SIZE 1024   

typedef struct MemBlock {
    int start;                
    int size;                 
    bool is_free;            
    struct MemBlock *prev;
    struct MemBlock *next;
} MemBlock;

static MemBlock *mem_head = NULL;   

void mem_init(void) {
    mem_head = (MemBlock*)malloc(sizeof(MemBlock));
    mem_head->start = 0;
    mem_head->size = MEM_POOL_SIZE;
    mem_head->is_free = true;
    mem_head->prev = NULL;
    mem_head->next = NULL;
    printf("内存池初始化，总大小：%d 字节\n", MEM_POOL_SIZE);
}

void mem_show(void) {
    printf("\n========== 内存状态 ==========\n");
    MemBlock *p = mem_head;
    while (p) {
        printf("地址 [%4d ~ %4d] 大小：%4d  %s\n",
               p->start, p->start + p->size - 1,
               p->size, p->is_free ? "空闲" : "已占用");
        p = p->next;
    }
    printf("================================\n\n");
}

int mem_alloc(int req_size) {
    if (req_size <= 0 || req_size > MEM_POOL_SIZE) {
        printf("分配失败：请求大小无效\n");
        return -1;
    }

    MemBlock *p = mem_head;
    while (p) {
        if (p->is_free && p->size >= req_size) {
            if (p->size > req_size) {
                MemBlock *new_free = (MemBlock*)malloc(sizeof(MemBlock));
                new_free->start = p->start + req_size;
                new_free->size = p->size - req_size;
                new_free->is_free = true;
                new_free->prev = p;
                new_free->next = p->next;

                if (p->next) p->next->prev = new_free;
                p->next = new_free;
                p->size = req_size;
            }
            p->is_free = false;
            printf("分配成功，起始地址：%d，大小：%d\n", p->start, p->size);
            return p->start;
        }
        p = p->next;
    }
    printf("分配失败：内存不足\n");
    return -1;
}

void mem_free(int start_addr) {
    MemBlock *p = mem_head;
    while (p) {
        if (p->start == start_addr) {
            if (p->is_free) {
                printf("警告：地址 %d 已经释放过，忽略\n", start_addr);
                return;
            }
            p->is_free = true;

            if (p->prev && p->prev->is_free) {
                MemBlock *prev = p->prev;
                prev->size += p->size;
                prev->next = p->next;
                if (p->next) p->next->prev = prev;
                free(p);
                p = prev;   
            }

            if (p->next && p->next->is_free) {
                MemBlock *next = p->next;
                p->size += next->size;
                p->next = next->next;
                if (next->next) next->next->prev = p;
                free(next);
            }

            printf("释放成功，起始地址：%d\n", start_addr);
            return;
        }
        p = p->next;
    }
    printf("释放失败：未找到起始地址 %d\n", start_addr);
}

void mem_compact(void) {
    int offset = 0;
    MemBlock *p = mem_head;
    MemBlock *new_head = NULL, *tail = NULL;

    while (p) {
        if (!p->is_free) {   
            MemBlock *node = (MemBlock*)malloc(sizeof(MemBlock));
            node->start = offset;
            node->size = p->size;
            node->is_free = false;
            node->prev = tail;
            node->next = NULL;

            if (!new_head) new_head = node;
            else tail->next = node;
            tail = node;

            offset += p->size;
        }
        MemBlock *old = p;
        p = p->next;
        free(old);   
    }

    if (offset < MEM_POOL_SIZE) {
        MemBlock *free_node = (MemBlock*)malloc(sizeof(MemBlock));
        free_node->start = offset;
        free_node->size = MEM_POOL_SIZE - offset;
        free_node->is_free = true;
        free_node->prev = tail;
        free_node->next = NULL;

        if (tail) tail->next = free_node;
        else new_head = free_node;   
        tail = free_node;
    }

    mem_head = new_head;
    printf("紧凑整理完成，已用 %d 字节，空闲 %d 字节\n", offset, MEM_POOL_SIZE - offset);
}

void mem_destroy(void) {
    MemBlock *p = mem_head;
    while (p) {
        MemBlock *tmp = p;
        p = p->next;
        free(tmp);
    }
    mem_head = NULL;
    printf("内存管理器已销毁\n");
}
