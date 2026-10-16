# Phase 2 — Debugging VTEST_CLIENT_ERROR_CONTEXT_FAILED (2026-08-27, autonomous round 5)

## Hypotheses ruled out

1. **Not a version mismatch.** virglrenderer and Mesa pull **the same commit** of the `venus-protocol` subproject (`e94b12f301b9eb27ebead757128a18420b4f7994`). This possibility is ruled out.

2. **Not missing AMD/RADV support (probably).** Web research: virglrenderer officially supports Venus with RADV 21.1+ (our RADV is much newer, Mesa 26.x). It is marked "experimental", but that is not the same as unsupported. (Note: a search turned up a statement like "AMDGPU native context support since 1.10", but our virglrenderer version is `1.3.0` — these numbers look inconsistent, the search summary is probably unreliable, so I ignored it.)

3. **Not a handshake/protocol-version desync.** The server code (`vtest_client_dispatch_commands`) returns `VTEST_CLIENT_ERROR_CONTEXT_MISSING` if the first command is not `VCMD_CREATE_RENDERER` — we got **`VTEST_CLIENT_ERROR_CONTEXT_FAILED`**, so the server recognized the first command correctly (as CREATE_RENDERER). This shows that the ping/protocol-version negotiation triggered by `VN_DEBUG=vtest` did not break the byte alignment.

## Error location pinned down

The error occurs in `vtest_create_context()` in `vtest/vtest_renderer.c:744` — this function can return -1 in only 2 places:
- `length > 1024*1024` (name too long) — impossible, the name is a short string.
- if the number of bytes returned by the `ctx->input->read(ctx->input, vtestname, length)` call is not equal to `length`.

So the server **cannot read the "renderer name" string sent by the client at exactly the
expected length.** The client side (`vn_renderer_vtest.c:170-181`,
`vtest_vcmd_create_renderer`) sends the header and the name with **two separate `write()`
calls**. Whether the server's read function correctly joins them (that is, whether it loops
properly over partial reads on the socket) has not been verified.

## Concrete Task for the Next Round
Runtime tracing is needed (static code reading was not enough):
1. Trace both the client and the server process with `strace -f -e trace=read,write -s 200` and compare how many bytes are actually sent and read during the CREATE_RENDERER command.
2. Alternative: add a temporary `fprintf(stderr, ...)` debug line to `vtest_create_context()` (print the `ret` value and `length`), rebuild, and see exactly what the read returns.
3. If it really is a partial-read problem, it may be a missing loop/edge case in virglrenderer's vtest read function (probably `vtest_block_read`, not read yet) — in that case, check the upstream bug tracker for a known issue.
