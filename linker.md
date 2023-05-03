# linker

[linker log](linker-log.md)

为什么 dl_iterator_phdr 可以遍历到 linker (第一个就是)，而 dladdr 不能根据 linker 的地址取得 linker 的 Dlinfo ？

bionic/linker/linker.cpp

两者同样遍历 solist 链表：

```cpp
// Here, we only have to provide a callback to iterate across all the
// loaded libraries. gcc_eh does the rest.
int do_dl_iterate_phdr(int (*cb)(dl_phdr_info* info, size_t size, void* data), void* data) {
  int rv = 0;
  for (soinfo* si = solist_get_head(); si != nullptr; si = si->next) {
    dl_phdr_info dl_info;
    dl_info.dlpi_addr = si->link_map_head.l_addr;
    dl_info.dlpi_name = si->link_map_head.l_name;
```

dl_iterator_phdr 的 dlpi_addr 取自 soinfo.link_map_head.l_addr

```cpp
int do_dladdr(const void* addr, Dl_info* info) {
  // Determine if this address can be found in any library currently mapped.
  soinfo* si = find_containing_library(addr);
  if (si == nullptr) {
    return 0;
  }

  memset(info, 0, sizeof(Dl_info));

  info->dli_fname = si->get_realpath();
  // Address at which the shared object is loaded.
  info->dli_fbase = reinterpret_cast<void*>(si->base);

  // Determine if any symbol in the library contains the specified address.
  ElfW(Sym)* sym = si->find_symbol_by_address(addr);
  if (sym != nullptr) {
    info->dli_sname = si->get_string(sym->st_name);
    info->dli_saddr = reinterpret_cast<void*>(si->resolve_symbol_address(sym));
  }

  return 1;
}
```

dladdr 判断地址是否位于 so 的某个 loadable segment 里面，用的是 soinfo.base 和 soinfo.size

find_containing_library:

```cpp
soinfo* find_containing_library(const void* p) {
  // Addresses within a library may be tagged if they point to globals. Untag
  // them so that the bounds check succeeds.
  ElfW(Addr) address = reinterpret_cast<ElfW(Addr)>(untag_address(p));
  for (soinfo* si = solist_get_head(); si != nullptr; si = si->next) {
    if (address < si->base || address - si->base >= si->size) {
      continue;
    }
    ElfW(Addr) vaddr = address - si->load_bias;
    for (size_t i = 0; i != si->phnum; ++i) {
      const ElfW(Phdr)* phdr = &si->phdr[i];
      if (phdr->p_type != PT_LOAD) {
        continue;
      }
      if (vaddr >= phdr->p_vaddr && vaddr < phdr->p_vaddr + phdr->p_memsz) {
        return si;
      }
    }
  }
  return nullptr;
}
```

而 linker_main 初始化自己的 soinfo 的时候似乎没有初始化这个字段，因此无法通过 find_containing_library 得到 linker 自身（不知道是不是有意而为的）。

```cpp
// bionic/linker/linker_main.cpp __linker_init_post_relocation
// bionic/linker/dlfcn.cpp get_libdl_info
```

同样的道理，__loader_dlopen 的 caller_addr 传 __loader_dlopen 自己的地址（也就是 linker 的地址）也不能让获取到的 ns 变成 linker 所属的 ns ，而是使用了 anonymous ns ，默认是 global ns ，不过在 java 进程会设置成 classloader ns 。
