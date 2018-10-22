###Ebtables 功能介绍

### Ebtables常见用法

### Ebtables 主要结构
```c
struct ebt_u_match* ebt_matches;
struct ebt_u_match
{
    char name[EBT_FUNCTION_MAXNAMELEN];
    /* size of the real match data */
    unsigned int size;
    void (*help)(void);
    void (*init)(struct ebt_entry_match *m);
    int (*parse)(int c, char **argv, int argc,
            const struct ebt_u_entry *entry, unsigned int *flags,
            struct ebt_entry_match **match);
    void (*final_check)(const struct ebt_u_entry *entry,
       const struct ebt_entry_match *match,
       const char *name, unsigned int hookmask, unsigned int time);
    void (*print)(const struct ebt_u_entry *entry,
       const struct ebt_entry_match *match);
    int (*compare)(const struct ebt_entry_match *m1,
       const struct ebt_entry_match *m2);
    const struct option *extra_ops;
    /*
     * can be used e.g. to check for multiple occurance of the same option
     */
    unsigned int flags;
    unsigned int option_offset;
    struct ebt_entry_match *m;
    /*
     * if used == 1 we no longer have to add it to
     * the match chain of the new entry
     * be sure to put it back on 0 when finished
     */
    unsigned int used;
    struct ebt_u_match *next;
};

struct ebt_entry_match
{
    union {
        char name[EBT_FUNCTION_MAXNAMELEN];
        struct ebt_match *match;
    } u;
    /* size of data */
    unsigned int match_size;
#ifdef KERNEL_64_USERSPACE_32
    unsigned int pad;
#endif
    unsigned char data[0] __attribute__ ((aligned (__alignof__(struct ebt_replace))));
}; 
```
### Ebtables 代码结构
#### Ebtables 初始化部分
Ebtables 链接了动态库，libebt_ip.so，libebt_ip6.so ， libebt_ivlan.so 等等。
这些动态库都有一个_init 函数，该函数会在main函数执行之前被调用。
```c
void _init(void)
{
    ebt_register_match(&ivlan_match);
}
 
void ebt_register_match(struct ebt_u_match* m)
{
    int size = EBT_ALIGN(m->size) + sizeof(struct ebt_entry_match);
    struct ebt_u_match** i;
    m->m = (struct ebt_entry_match*)malloc(size);
    if (!m->m)
    {
        ebt_print_memory();
    }
    strcpy(m->m->u.name, m->name);
    m->m->match_size = EBT_ALIGN(m->size);
    m->init(m->m);

    for (i = &ebt_matches; *i; i = &((*i)->next));
    m->next = NULL;
    *i = m;
}
   
```
**Ebtables Entry  ebtables-standalone.c**
```c
int main(int argc, char *argv[])
{
    ebt_silent = 0;                                                                                                                                                                                               
    ebt_early_init_once();
    strcpy(replace.name, "filter");  // Default table is filter table.
    do_command(argc, argv, EXEC_STYLE_PRG, &replace);
    return 0;
}

void ebt_early_init_once()                                                                                                                                                                                        
{
    ebt_iterate_matches(merge_match);
    ebt_iterate_watchers(merge_watcher);
    ebt_iterate_targets(merge_target);
}

void ebt_iterate_matches(void (*f)(struct ebt_u_match*))
{
    struct ebt_u_match* i;

    for (i = ebt_matches; i; i = i->next)
    {                                                                                                                                                                                                               
        f(i);
    }    
}
static void merge_match(struct ebt_u_match *m)
{
    ebt_options = merge_options
       (ebt_options, m->extra_ops, &(m->option_offset));
}
static struct option *merge_options(struct option *oldopts,
   const struct option *newopts, unsigned int *options_offset)
{
    unsigned int num_old, num_new, i;
    struct option *merge;

    if (!newopts || !oldopts || !options_offset)
        ebt_print_bug("merge wrong");
    for (num_old = 0; oldopts[num_old].name; num_old++);
    for (num_new = 0; newopts[num_new].name; num_new++);
    global_option_offset += OPTION_OFFSET;
    *options_offset = global_option_offset;
    
    merge = malloc(sizeof(struct option) * (num_new + num_old + 1));
    if (!merge)
        ebt_print_memory();
    memcpy(merge, oldopts, num_old * sizeof(struct option));
    for (i = 0; i < num_new; i++) {
        merge[num_old + i] = newopts[i];
        merge[num_old + i].val += *options_offset;
    }
    memset(merge + num_old + num_new, 0, sizeof(struct option));
    /* Only free dynamically allocated stuff */
    if (oldopts != ebt_original_options)
    {
        printf("Johnson Free\n");
        free(oldopts);
    }
    return merge;
}
```
#### Ebtables Process Command
```c
int do_command(int argc, char *argv[], int exec_style,
               struct ebt_u_replace *replace_)
{
        while ((c = getopt_long(argc, argv,
       "-A:D:C:I:N:E:X::L::Z::F::P:Vhi:o:j:c:p:s:d:t:M:", ebt_options, NULL)) != -1) {
            switch(c)
            {

                
            }
        }
    
}
```
```c
int check_and_change_rule_number
{
    ...
    ebt_check_rule_exists(replace, new_entry);
}
int ebt_check_rule_exists(struct ebt_u_replace *replace,
              struct ebt_u_entry *new_entry)
{
    ...
    while (m_l) {
        m = (struct ebt_u_match *)(m_l->m);
        m_l2 = u_e->m_list;
        while (m_l2 && strcmp(m_l2->m->u.name, m->m->u.name))
            m_l2 = m_l2->next;
        if (!m_l2 || !m->compare(m->m, m_l2->m))
            goto letscontinue;
        j++;
        m_l = m_l->next;
    }

}
```