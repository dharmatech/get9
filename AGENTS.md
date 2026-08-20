# Get9 agent guidance

- Route VM lifecycle work through the `p9qemu` skill and Drawterm transport
  through the `plan9-drawterm` skill. Use the adjacent Plan 9 skills for guest
  file search, documentation, source, Acme, and git9 work.
- The Linux source root is `/home/dharmatech/src/get9`; the project VM root is
  `/home/dharmatech/vm/get9`; and `dev` is the ordinary mutable instance.
- Treat `checkpoint-000` and `checkpoint-001-ipv6-lookups-off` as frozen
  baselines. Never boot or modify them. Copy a halted checkpoint to a distinct
  writable instance before using it.
- Prefer a narrow Drawterm export of the Get9 source directory for guest
  transfer. Use `$home/src/get9` as the ordinary guest-side source/test copy.
- Keep recipes directly executable and independently inspectable. Preserve the
  versioned, user-local installation model unless a recipe explicitly documents
  a different method.
