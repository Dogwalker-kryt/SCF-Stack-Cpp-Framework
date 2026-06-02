# SCF String Library - Documentation

## Overview

**SCF String** (`scf_str.hpp`) is a lightweight, stack-allocated fixed-size string class designed for performance-critical and embedded C++17 applications. It provides a **zero-heap** alternative to `std::string` with a familiar API while maintaining strict compile-time bounds checking.

### Key Philosophy

- **No heap allocation** — entire string fits on the stack
- **No dependencies** — uses only standard library headers
- **Performance-focused** — minimal overhead, cache-friendly
- **Drop-in compatible** — seamless interop with `std::string`
- **Learning/showoff** — demonstrates modern C++ template techniques

---

## Core Class: `fxdstr<N>`

### Template Parameters

- **`N`** — Maximum string length in characters (excluding null terminator)
- String buffer: `std::array<char, N + 1>` with null terminator

### Example

```cpp
scf::str256 device = "/dev/sda";           // 256-char fixed string
scf::str32 fstype = "ext4";                // 32-char fixed string
scf::str1024 config;                       // 1024-char fixed string
```

---

## Type Aliases

Pre-defined aliases for common sizes in `scf` namespace:

```cpp
using str8    = fxdstr<8>;       // tiny strings: flags, single chars
using str16   = fxdstr<16>;      // short strings: types, names
using str32   = fxdstr<32>;      // device names, short paths
using str64   = fxdstr<64>;      // UUIDs, identifiers
using str128  = fxdstr<128>;     // mount points, file metadata
using str256  = fxdstr<256>;     // general purpose, paths
using str512  = fxdstr<512>;     // command output, buffers
using str1024 = fxdstr<1024>;    // large configs, formatted text
using str2048 = fxdstr<2048>;    // extended buffers
using str4096 = fxdstr<4096>;    // very large strings

// System-dependent default:
using str_t = fxdstr<256>;       // 64-bit default
// or
using str_t = fxdstr<128>;       // 32-bit default
```

---

## Constructors & Assignment

### Constructors

```cpp
scf::str256 s1;                              // empty
scf::str256 s2("hello");                    // from C-string
scf::str256 s3(5, 'x');                     // 5 x's: "xxxxx"

scf::str256 s4(std::string("world"));       // from std::string
scf::str256 s5(std::string_view("view"));   // from string_view

scf::str128 s6(s1);                         // copy construct
scf::str512 s7(s6);                         // cross-size construct (throws if too big)
```

### Assignment

```cpp
scf::str256 s;
s = "hello";                      // from C-string
s = std::string("world");         // from std::string
s = std::string_view("view");     // from string_view
s = 'x';                          // single char
s = s1;                           // copy from another fxdstr
```

### Exception Safety

All operations throw `std::length_error` if the result exceeds capacity `N`:

```cpp
scf::str8 tiny;
try {
    tiny = "this string is way too long for 8 chars";  // throws
} catch (const std::length_error& e) {
    std::cerr << "String overflow: " << e.what() << std::endl;
}
```

---

## String Operations

### Basic Access

```cpp
scf::str256 s = "hello";

// Element access
char c = s[0];                    // 'h' (unchecked)
char d = s.at(0);                // 'h' (throws if out of range)
char first = s.front();           // 'h'
char last = s.back();             // 'o'

// C interop
const char* ptr = s.c_str();      // null-terminated
char* data = s.data();            // mutable buffer
```

### Capacity

```cpp
scf::str32 s = "abc";

s.size();          // 3
s.length();        // 3
s.max_size();      // 32 (capacity)
s.capacity();      // 32
s.empty();         // false
```

### Modification

```cpp
scf::str256 s;

// Append
s.append("hello");                // append C-string
s.append(3, '!');                 // append 3 '!'s
s += " world";                    // operator+=
s.push_back('x');                 // single char

// Erase
s.erase(5);                       // delete from position 5 to end
s.erase(0, 3);                    // delete 3 chars starting at 0
s.pop_back();                      // remove last char

// Replace
s.replace(0, 5, "goodbye");       // replace 5 chars at 0

// Clear
s.clear();                        // empty the string
```

---

## Search & Substring

### Search Operations

```cpp
scf::str256 s = "hello world";

// Find
size_t pos = s.find("world");              // 6
size_t pos2 = s.find('l');                 // 2
size_t pos3 = s.find(std::string("o"));    // 4

// Reverse find
size_t rpos = s.rfind("o");                // 7
size_t rpos2 = s.rfind('l', 5);            // 3 (search backwards from pos 5)

// No match
if (s.find("xyz") == s.npos) {
    std::cout << "Not found\n";
}
```

### Predicates (C++20 style)

```cpp
scf::str256 s = "hello";

bool b1 = s.starts_with("he");    // true
bool b2 = s.starts_with('h');     // true
bool b3 = s.ends_with("lo");      // true
bool b4 = s.ends_with('o');       // true
```

### Substring

```cpp
scf::str256 s = "hello world";

scf::str256 sub1 = s.substr(0, 5);        // "hello"
scf::str32 sub2 = s.substr_sized<32>(6);  // "world" (smart-size)

// substr_sized throws if result too big for target:
try {
    scf::str4 sub3 = s.substr_sized<4>(0, 5);  // throws (5 > 4)
} catch (const std::length_error& e) { }
```

---

## Comparison

```cpp
scf::str256 a = "apple";
scf::str256 b = "banana";

// Operators
bool eq = (a == b);               // false
bool ne = (a != b);               // true
bool lt = (a < b);                // true
bool le = (a <= b);               // true
bool gt = (a > b);                // false
bool ge = (a >= b);               // false

// Manual compare
int cmp = a.compare(b);           // negative (a < b)
int cmp2 = a.compare("apple");    // 0 (equal)
```

---

## Concatenation

### fxdstr + fxdstr

```cpp
scf::str256 a = "hello";
scf::str256 b = "world";
auto result = a + b;              // type: fxdstr<512>
```

### fxdstr + Literals

```cpp
scf::str256 a = "hello";
auto result1 = a + " world";      // type: fxdstr<N + literal_len>
auto result2 = "prefix: " + a;    // type: fxdstr<literal_len + N>
auto result3 = a + '!';           // type: fxdstr<257> (one larger)
```

### Conversion with std::string

```cpp
scf::str256 s = "hello";

// Implicit conversion
std::string str1 = s;            // automatic operator std::string()
std::string_view sv = s;          // automatic operator string_view()

// Explicit conversion
std::string str2 = s.to_std_str();
```

### Stream Output

```cpp
scf::str256 s = "hello";

std::cout << s << std::endl;      // works like std::string
std::string str = scf::to_str64(s);  // convert to smaller size
```

---

## Conversion Functions

### `to_scf_str<N>(value)` — Generic Converter

Converts any type to `fxdstr<N>`:

```cpp
// From std::string
scf::str256 s1 = scf::to_scf_str<256>(std::string("hello"));

// From arithmetic types (int, long, float, etc.)
scf::str32 s2 = scf::to_scf_str<32>(42);          // "42"
scf::str64 s3 = scf::to_scf_str<64>(3.14159);     // "3.141590"
scf::str16 s4 = scf::to_scf_str<16>(true);        // "true"

// Cross-size fxdstr conversion
scf::str256 big = "hello";
scf::str32 small = scf::to_scf_str<32>(big);      // "hello"
// throws if source > 32: scf::to_scf_str<8>(big);
```

### Convenience Aliases

```cpp
// All do the same thing, but specify the target size:
scf::str8 s1 = scf::to_str8(42);
scf::str16 s2 = scf::to_str16(3.14);
scf::str256 s3 = scf::to_str256(std::string("x"));
scf::str1024 s4 = scf::to_str1024(some_value);
```

### Auto-Sized Conversion

```cpp
auto s = scf::to_str(42);         // type: fxdstr<256>
auto s2 = scf::to_str(true);      // type: fxdstr<256>
```

### Supported Types

| Type | Example |
|------|---------|
| `int`, `unsigned int` | `42`, `-5` |
| `long`, `unsigned long` | `1000000L` |
| `long long`, `unsigned long long` | `9223372036854775807LL` |
| `float`, `double`, `long double` | `3.14`, `2.71828` |
| `char`, `unsigned char`, `short`, `unsigned short` | `'x'`, `65` |
| `bool` | `true` (→ `"true"`) |
| `std::string` | `std::string("text")` |
| `std::string_view` | `std::string_view("view")` |
| `const char*` | `"hello"` |
| `fxdstr<M>` (any size) | `fxdstr<256>("x")` → `str64` |

---

## Memory & Low-Level

### Direct Buffer Access (Unsafe)

```cpp
scf::str256 s;

// Get mutable buffer pointer
char* buf = s.data();

// Perform low-level operations
strcpy(buf, "hello");             // ⚠️ Caller responsible for bounds

// Set length after manual buffer modification
s.set_length(5);                  // Update internal length tracker
s.set_length(0);                  // Throws if > capacity

// Direct length reference (dangerous!)
size_t& len_ref = s.length_ref(); // ⚠️ Only for advanced use
```

### Stack Footprint

```cpp
scf::str8 tiny;      // 9 bytes (8 + 1 null + size tracking)
scf::str256 medium;  // ~264 bytes
scf::str4096 huge;   // ~4100 bytes (stack allocation!)
```

**Warning:** `str4096` and larger allocate significant stack space. Use sparingly or on threads with large stacks.

---

## API Reference Summary

### Capacity/Info
| Method | Returns | Notes |
|--------|---------|-------|
| `size()` | `size_t` | current length |
| `length()` | `size_t` | same as size() |
| `max_size()` | `size_t` | template parameter N |
| `capacity()` | `size_t` | same as max_size() |
| `empty()` | `bool` | true if size == 0 |

### Access
| Method | Returns | Notes |
|--------|---------|-------|
| `operator[](i)` | `char&` | unchecked |
| `at(i)` | `char&` | throws if i >= len |
| `front()` | `char&` | first char |
| `back()` | `char&` | last char |
| `c_str()` | `const char*` | null-terminated |
| `data()` | `char*` | mutable buffer |

### Modification
| Method | Effect | Throws |
|--------|--------|--------|
| `append(str)` | add to end | if overflow |
| `push_back(c)` | add char | if overflow |
| `pop_back()` | remove last | if empty |
| `erase(pos, count)` | delete range | if out of range |
| `replace(pos, count, str)` | replace range | if result too big |
| `clear()` | empty string | never |

### Search
| Method | Returns | Notes |
|--------|---------|-------|
| `find(str, pos)` | `size_t` | npos if not found |
| `rfind(str, pos)` | `size_t` | reverse find |
| `starts_with(str)` | `bool` | C++20 style |
| `ends_with(str)` | `bool` | C++20 style |

### Conversion
| Method | Type | Notes |
|--------|------|-------|
| `operator std::string()` | implicit | automatic |
| `operator std::string_view()` | implicit | automatic |
| `to_std_str()` | `std::string` | explicit |

---

## Design Patterns & Best Practices

### Pattern 1: Sized-Appropriate Strings

```cpp
struct DiskInfo {
    scf::str32 device;           // /dev/sda, /dev/nvme0n1p1
    scf::str16 fstype;           // ext4, ntfs, xfs
    scf::str128 mount_point;     // /mnt/usb, /home/user
    scf::str512 model;           // Samsung SSD 970 EVO 1TB
};
```

### Pattern 2: Configuration with fxdstr

```cpp
struct Config {
    scf::str256 config_path;
    scf::str8 theme;             // "light", "dark"
    scf::str32 log_level;        // "debug", "info", "warn", "error"
    bool enabled = true;
};
```

### Pattern 3: Building Strings

```cpp
scf::str1024 cmd;
cmd = "dd if=" + drive + " of=/tmp/backup.img bs=4M";
EXEC(cmd);  // execute command

// Better: use append for clarity
cmd.clear();
cmd.append("mount ");
cmd.append(device);
cmd.append(" /mnt/usb");
```

### Pattern 4: Bounds Checking at Compile-Time

```cpp
// Compile-time failure if concatenation impossible
// auto result = scf::str8("hello") + scf::str8("world");  // error! 8+8=16 needed

// Safe: use larger target
auto result = scf::str256("hello") + scf::str256("world");  // ok
```

### Pattern 5: Mixed std::string / fxdstr

```cpp
// Safe interop - no conversion errors
scf::str256 bounded = "device";
std::string dynamic = "path";

// Concatenation works both ways
auto cmd = bounded + " " + dynamic;       // → std::string (implicit)
auto result = dynamic + bounded.c_str();  // → std::string
```

---

## Performance Notes

### Advantages

✅ **Zero heap allocation** — all data on stack  
✅ **Cache-friendly** — contiguous memory  
✅ **Compile-time sizing** — no runtime bounds checks needed  
✅ **Fast copying** — array copy vs string allocation  
✅ **No ABA problems** — pointer never changes  
✅ **Exception-safe** — single allocation never fails  

### Limitations

❌ **Stack space** — `str4096` uses 4KB per instance  
❌ **Fixed size** — can't grow beyond N  
❌ **No small-string optimization** — always uses full N bytes  
❌ **Cross-template operations** — `str32 + str64` requires explicit sizing  

### When to Use

| Use Case | Recommendation |
|----------|---|
| Device names, identifiers | ✅ Use `str32`–`str64` |
| Config values, flags | ✅ Use `str8`–`str32` |
| Mount points, paths | ✅ Use `str128`–`str256` |
| Command execution | ⚠️ Depends on cmd length; prefer `std::string` |
| User input, log messages | ⚠️ Often too dynamic; prefer `std::string` |
| Loop-heavy hardware enumeration | ✅ Use appropriate size |

---

## Integration Example

### Complete Usage

```cpp
#include "scf_str.hpp"

int main() {
    using namespace scf;

    // Fixed metadata
    str32 device = "/dev/sda1";
    str16 fstype = "ext4";
    str128 mount_pt = "/mnt/backup";

    // Build command
    str512 cmd;
    cmd.append("mount -t ");
    cmd.append(fstype);
    cmd.append(" ");
    cmd.append(device);
    cmd.append(" ");
    cmd.append(mount_pt);

    std::cout << "Executing: " << cmd << std::endl;

    // Conversion
    str64 uuid = to_str64("a1b2c3d4-e5f6-47a8-b9c0-d1e2f3a4b5c6");
    
    // Safe cross-size with explicit bounds
    str32 short_uuid = uuid.substr_sized<32>(0, 8);  // "a1b2c3d4"

    // Safe interop
    std::string full_path = "/mnt/" + device;        // works!

    return 0;
}
```

---

## License & Attribution

This library is part of the **SCF (Stack C++ Framework)** project and is designed for educational and commercial use.

**Author:** Dogwalker-kryt  
**Standard:** C++17  
**Dependencies:** Standard library only  

---

## FAQ

### Q: Can I use fxdstr in containers?

**A:** Yes! Full support:
```cpp
std::vector<scf::str256> devices;
std::unordered_map<scf::str32, scf::str256> device_map;
std::set<scf::str64> uuids;
```

### Q: What happens if I exceed capacity?

**A:** All operations throw `std::length_error`:
```cpp
try {
    scf::str8 s = "way too long string";
} catch (const std::length_error& e) {
    // Handle overflow
}
```

### Q: Can I interop with C functions expecting const char*?

**A:** Absolutely:
```cpp
scf::str256 path = "/etc/passwd";
FILE* f = fopen(path.c_str(), "r");  // works!
```

### Q: Performance vs std::string?

**A:** Generally **faster** for bounded strings due to stack allocation, but trade-off is fixed size. Not suitable for highly dynamic strings.

### Q: Can I iterate?

**A:** Yes, full iterator support:
```cpp
scf::str256 s = "hello";
for (auto it = s.begin(); it != s.end(); ++it) {
    std::cout << *it;
}
```

---

## Summary

`fxdstr<N>` is a **production-ready, high-performance fixed string class** for C++17 applications that need bounded, stack-allocated strings. It combines the familiarity of `std::string` with the performance and determinism of fixed-size buffers.

Use it for **device metadata, configuration, identifiers, and loop-critical string operations**. Keep `std::string` for truly dynamic content.
