# U1 spec — ring buffer port (behavior, not structure)

Source: `reference/gnet-go/pkg/buffer/ring/ring_buffer.go` (gnet v2.10.0, Apache-2.0)
Deps to port first (same unit): `pkg/math/math.go` (CeilToPowerOfTwo etc.), `pkg/pool/byteslice` (replace with plain Vec — pool is an optimization, not behavior).

## Rust API (crates/gnet-core, module `ring`)

```rust
pub const MIN_READ: usize = 512;
pub const DEFAULT_BUFFER_SIZE: usize = 1024;
const BUFFER_GROW_THRESHOLD: usize = 4096;

pub struct RingBuffer { /* buf: Vec<u8>, size: usize, r: usize, w: usize, is_empty: bool */ }

impl RingBuffer {
    pub fn new(size: usize) -> Self;               // size==0 -> empty buf; else ceil_pow2
    pub fn peek(&self, n: isize) -> (Vec<u8>, Vec<u8>); // head,tail; n<=0 -> all
    pub fn discard(&mut self, n: isize) -> usize;  // returns discarded count
    pub fn read(&mut self, p: &mut [u8]) -> Result<usize, RingError>; // Err(Empty) if empty & p nonempty
    pub fn read_byte(&mut self) -> Result<u8, RingError>;
    pub fn write(&mut self, p: &[u8]) -> usize;    // grows as needed
    pub fn write_byte(&mut self, c: u8);
    pub fn write_string(&mut self, s: &str) -> usize;
    pub fn buffered(&self) -> usize;
    pub fn len(&self) -> usize;    // underlying buf len (Go Len() = len(buf))
    pub fn cap(&self) -> usize;    // == size
    pub fn available(&self) -> usize;
    pub fn bytes(&self) -> Vec<u8>; // copy of readable bytes
    pub fn is_full(&self) -> bool;  // r==w && !is_empty
    pub fn is_empty(&self) -> bool;
    pub fn reset(&mut self);
    pub fn read_from<R: std::io::Read>(&mut self, r: &mut R) -> std::io::Result<u64>;
    pub fn write_to<W: std::io::Write>(&mut self, w: &mut W) -> std::io::Result<u64>;
}

pub enum RingError { Empty }
```

## Behavior pins (must match Go exactly)

- `new(0)`: is_empty=true, buf empty; first `write` triggers grow to DEFAULT_BUFFER_SIZE.
- `CeilToPowerOfTwo`: n<=2 -> 2; else next pow2 of (n-1) bits.Len. Port `math.go` faithfully including its edge cases (n<=2 returns 2 — note Go returns 2 even for n=0/negative path via New only calling it when size!=0).
- `Buffered()`: r==w && is_empty -> 0; r==w && !is_empty -> size; w>r -> w-r; else size-r+w.
- `Available()`: r==w && is_empty -> size; r==w && !is_empty -> 0; w<r -> r-w; else size-w+r.
- `grow(newCap)`: exact algorithm from Go lines 487-522 — doubleCap rule, bufferGrowThreshold 4KB branch, n += n/4 loop. Port faithfully; byteslice pool -> Vec::with_capacity (no unsafe pool).
- `Peek(n)`: n<=0 returns ALL readable bytes as (head,tail) — Go callers use Peek(-1) for full contents. head/tail split at wrap point.
- `Discard(n)`: n<=0 -> 0; n<buffered -> advance r=(r+n)%size return n; else Reset and return buffered.
- `Read`: empty+nonempty p -> Err(Empty); copies min(len(p), buffered); resets when r catches w.
- `ReadFrom`: Go's loop reads into buf[w:] then buf[:r] when w>=r — port the same two-read pattern per iteration; grow when Available<MinRead via grow(buffered+MinRead). io.EOF -> Ok(n).
- `WriteTo`: Go writes contiguous head then tail; ErrShortWrite semantics — in Rust return Ok(bytes_written) and let caller see buffer empty/non-empty; document difference.
- `Bytes()`: returns concatenated readable bytes (head+tail), empty vec when is_empty.
- WriteByte grows by grow(1) when Available<1 — Go's grow(1) with size>0 doubles.

## Tests (port the 7 Go tests, same assertions)

TestRingBuffer_Write, TestZeroRingBuffer, TestRingBufferGrow, TestRingBuffer_Read, TestRingBuffer_ByteInterface, TestRingBuffer_ReadFrom, TestRingBuffer_WriteTo — deterministic parts verbatim; crypto/rand parts use fixed-seed bytes (e.g. a simple xorshift or repeated pattern) since Rust test can't call crypto/rand the same way — assertions on lengths/cap/buffered stay identical.

## Gates for this unit

cargo fmt --check; cargo clippy --workspace --all-targets -- -D warnings; cargo test --workspace; cargo doc --workspace --no-deps (zero warnings); comparator self-check (unit test asserting peek/discard/bytes invariants); real differential vs Go (see verification/).
