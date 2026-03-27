```c++
bool should_check_perms() const {
    return (is_fuse && !fuse_default_permissions) || (!is_fuse && client_permissions);
  }
```