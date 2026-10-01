# หลักสูตร Zig Programming - ตั้งแต่พื้นฐานจนถึงระดับโลก

> ภาษา Zig: Systems Programming Language ทดแทน C สำหรับยุคใหม่

---

## เกี่ยวกับหลักสูตรนี้

หลักสูตรนี้ออกแบบมาเพื่อสอน **Zig Programming Language** ตั้งแต่ระดับเริ่มต้นจนถึงระดับมืออาชีพและระดับโลก
โดยครอบคลุม 1000 ขั้นตอน แบ่งเป็น 100+ ส่วน (Parts) แต่ละส่วนมีเนื้อหาที่ใช้งานได้จริง

### ทำไมต้องเรียน Zig?

- **ประสิทธิภาพสูง**: ทำงานได้เร็วเทียบเท่า C/C++
- **ความปลอดภัย**: ไม่มี hidden memory allocation, ไม่มี hidden control flow
- **ง่ายกว่า C**: Syntax ที่สะอาดกว่า, error handling ที่ดีกว่า
- **Comptime**: compile-time computation ที่ทรงพลัง
- **Interop กับ C**: ใช้ C libraries ได้โดยตรง ไม่ต้อง FFI
- **Cross Compilation**: Build สำหรับ platform อื่นได้ง่าย
- **No Hidden Allocations**: รู้ทุก memory allocation
- **Modern Toolchain**: Build system ในตัว, Package manager

### โครงสร้างหลักสูตร

| ระดับ | Parts | Steps | เนื้อหา |
|-------|-------|-------|---------|
| พื้นฐาน | 1-20 | 1-200 | Syntax, Types, Functions, Memory |
| กลาง | 21-50 | 201-500 | Standard Library, Testing, Build System |
| สูง | 51-75 | 501-750 | Advanced Features, Optimization, C Interop |
| มืออาชีพ | 76-100+ | 751-1000+ | Projects, Patterns, Production |

---

## สารบัญ (Table of Contents)

### ระดับพื้นฐาน (Steps 1-200)

| Part | หัวข้อ | Steps |
|------|--------|-------|
| [Part 01](part-001-installation-and-setup.md) | การติดตั้งและเริ่มต้น | 1-10 |
| [Part 02](part-002-hello-world.md) | Hello World และโครงสร้างโปรแกรม | 11-20 |
| [Part 03](part-003-variables-and-constants.md) | ตัวแปรและค่าคงที่ | 21-30 |
| [Part 04](part-004-primitive-types.md) | ชนิดข้อมูลพื้นฐาน | 31-40 |
| [Part 05](part-005-operators.md) | ตัวดำเนินการ | 41-50 |
| [Part 06](part-006-control-flow.md) | การควบคุมการทำงาน | 51-60 |
| [Part 07](part-007-loops.md) | การวนลูป | 61-70 |
| [Part 08](part-008-functions.md) | ฟังก์ชัน | 71-80 |
| [Part 09](part-009-arrays-and-slices.md) | Arrays และ Slices | 81-90 |
| [Part 10](part-010-strings.md) | Strings และการจัดการข้อความ | 91-100 |
| [Part 11](part-011-structs.md) | Structs | 101-110 |
| [Part 12](part-012-enums.md) | Enums | 111-120 |
| [Part 13](part-013-unions.md) | Unions | 121-130 |
| [Part 14](part-014-pointers-basic.md) | Pointers พื้นฐาน | 131-140 |
| [Part 15](part-015-pointers-advanced.md) | Pointers ขั้นสูง | 141-150 |
| [Part 16](part-016-memory-management.md) | Memory Management | 151-160 |
| [Part 17](part-017-allocators.md) | Allocators | 161-170 |
| [Part 18](part-018-error-handling-basic.md) | Error Handling พื้นฐาน | 171-180 |
| [Part 19](part-019-error-handling-advanced.md) | Error Handling ขั้นสูง | 181-190 |
| [Part 20](part-020-optionals.md) | Optionals | 191-200 |

### ระดับกลาง (Steps 201-500)

| Part | หัวข้อ | Steps |
|------|--------|-------|
| [Part 21](part-021-comptime-basic.md) | Comptime พื้นฐาน | 201-210 |
| [Part 22](part-022-comptime-advanced.md) | Comptime ขั้นสูง | 211-220 |
| [Part 23](part-023-generics.md) | Generics | 221-230 |
| [Part 24](part-024-type-system.md) | Type System | 231-240 |
| [Part 25](part-025-interfaces.md) | Interfaces ด้วย Comptime | 241-250 |
| [Part 26](part-026-stdlib-overview.md) | Standard Library - ภาพรวม | 251-260 |
| [Part 27](part-027-stdlib-mem.md) | std.mem | 261-270 |
| [Part 28](part-028-stdlib-fmt.md) | std.fmt | 271-280 |
| [Part 29](part-029-stdlib-io.md) | std.io | 281-290 |
| [Part 30](part-030-stdlib-fs.md) | std.fs - File System | 291-300 |
| [Part 31](part-031-stdlib-os.md) | std.os - OS Interface | 301-310 |
| [Part 32](part-032-stdlib-process.md) | std.process | 311-320 |
| [Part 33](part-033-stdlib-math.md) | std.math | 321-330 |
| [Part 34](part-034-stdlib-sort.md) | std.sort | 331-340 |
| [Part 35](part-035-hashmap.md) | std.hash_map | 341-350 |
| [Part 36](part-036-arraylist.md) | std.ArrayList | 351-360 |
| [Part 37](part-037-string-handling.md) | String Handling ขั้นสูง | 361-370 |
| [Part 38](part-038-json.md) | std.json | 371-380 |
| [Part 39](part-039-testing.md) | std.testing | 381-390 |
| [Part 40](part-040-debugging.md) | Debugging | 391-400 |
| [Part 41](part-041-build-system-basic.md) | Build System พื้นฐาน | 401-410 |
| [Part 42](part-042-build-system-advanced.md) | Build System ขั้นสูง | 411-420 |
| [Part 43](part-043-c-interop-basic.md) | C Interoperability พื้นฐาน | 421-430 |
| [Part 44](part-044-c-interop-advanced.md) | C Interoperability ขั้นสูง | 431-440 |
| [Part 45](part-045-async.md) | Async/Coroutines | 441-450 |
| [Part 46](part-046-multithreading.md) | Multithreading | 451-460 |
| [Part 47](part-047-networking-basic.md) | Networking พื้นฐาน | 461-470 |
| [Part 48](part-048-http-server.md) | HTTP Server | 471-480 |
| [Part 49](part-049-tcp-udp.md) | TCP/UDP Programming | 481-490 |
| [Part 50](part-050-websocket.md) | WebSocket | 491-500 |

### ระดับสูง (Steps 501-750)

| Part | หัวข้อ | Steps |
|------|--------|-------|
| [Part 51](part-051-performance.md) | Performance Optimization | 501-510 |
| [Part 52](part-052-simd.md) | SIMD และ Vector Operations | 511-520 |
| [Part 53](part-053-inline-assembly.md) | Inline Assembly | 521-530 |
| [Part 54](part-054-embedded.md) | Embedded Systems | 531-540 |
| [Part 55](part-055-wasm.md) | WebAssembly | 541-550 |
| [Part 56](part-056-cross-compilation.md) | Cross Compilation | 551-560 |
| [Part 57](part-057-linking.md) | Linking และ Libraries | 561-570 |
| [Part 58](part-058-debugging-tools.md) | Debugging Tools | 571-580 |
| [Part 59](part-059-profiling.md) | Profiling | 581-590 |
| [Part 60](part-060-security.md) | Security Best Practices | 591-600 |
| [Part 61](part-061-project-cli.md) | Project: CLI Application | 601-610 |
| [Part 62](part-062-project-http.md) | Project: HTTP Server | 611-620 |
| [Part 63](part-063-project-filesystem.md) | Project: File System Tool | 621-630 |
| [Part 64](part-064-project-database.md) | Project: SQLite Wrapper | 631-640 |
| [Part 65](part-065-project-ssg.md) | Project: Static Site Generator | 641-650 |
| [Part 66](part-066-project-kvstore.md) | Project: Key-Value Store | 651-660 |
| [Part 67](part-067-project-scraper.md) | Project: Web Scraper | 661-670 |
| [Part 68](part-068-project-image.md) | Project: Image Processing | 671-680 |
| [Part 69](part-069-project-language.md) | Project: Calculator Language | 681-690 |
| [Part 70](part-070-project-shell.md) | Project: Shell Implementation | 691-700 |
| [Part 71](part-071-memory-patterns.md) | Advanced Memory Patterns | 701-710 |
| [Part 72](part-072-custom-allocators.md) | Custom Allocators | 711-720 |
| [Part 73](part-073-arena-allocator.md) | Arena Allocator Pattern | 721-730 |
| [Part 74](part-074-object-pool.md) | Object Pool Pattern | 731-740 |
| [Part 75](part-075-reference-counting.md) | Reference Counting | 741-750 |

### ระดับมืออาชีพ (Steps 751-1000+)

| Part | หัวข้อ | Steps |
|------|--------|-------|
| [Part 76](part-076-design-patterns.md) | Design Patterns ใน Zig | 751-760 |
| [Part 77](part-077-linked-list.md) | Data Structures: Linked List | 761-770 |
| [Part 78](part-078-binary-tree.md) | Data Structures: Binary Tree | 771-780 |
| [Part 79](part-079-graph.md) | Data Structures: Graph | 781-790 |
| [Part 80](part-080-hash-table.md) | Data Structures: Hash Table | 791-800 |
| [Part 81](part-081-sorting.md) | Algorithms: Sorting | 801-810 |
| [Part 82](part-082-searching.md) | Algorithms: Searching | 811-820 |
| [Part 83](part-083-dynamic-programming.md) | Algorithms: Dynamic Programming | 821-830 |
| [Part 84](part-084-graph-algorithms.md) | Algorithms: Graph Algorithms | 831-840 |
| [Part 85](part-085-lexer.md) | Parsing: Lexer | 841-850 |
| [Part 86](part-086-parser.md) | Parsing: Parser | 851-860 |
| [Part 87](part-087-ast.md) | Parsing: AST | 861-870 |
| [Part 88](part-088-compiler.md) | Compiler Basics | 871-880 |
| [Part 89](part-089-interpreter.md) | Interpreter Implementation | 881-890 |
| [Part 90](part-090-plugin-system.md) | Plugin System | 891-900 |
| [Part 91](part-091-dsl.md) | DSL Implementation | 901-910 |
| [Part 92](part-092-metaprogramming.md) | Metaprogramming | 911-920 |
| [Part 93](part-093-reflection.md) | Reflection | 921-930 |
| [Part 94](part-094-code-generation.md) | Code Generation | 931-940 |
| [Part 95](part-095-testing-strategies.md) | Testing Strategies | 941-950 |
| [Part 96](part-096-benchmarking.md) | Benchmarking | 951-960 |
| [Part 97](part-097-documentation.md) | Documentation | 961-970 |
| [Part 98](part-098-package-management.md) | Package Management | 971-980 |
| [Part 99](part-099-open-source.md) | Open Source Contribution | 981-990 |
| [Part 100](part-100-production.md) | Production Best Practices | 991-1000 |

---

## วิธีใช้หลักสูตรนี้

1. **เรียนตามลำดับ** - เริ่มจาก Part 01 และเรียนไปตามลำดับ
2. **ทดลองทำตาม** - เขียนโค้ดทุกตัวอย่างด้วยตัวเอง
3. **ทำแบบฝึกหัด** - แต่ละ Part มีแบบฝึกหัด ควรทำทุกข้อ
4. **สร้าง Project** - เมื่อถึง Part ที่มี Project ลองสร้างด้วยตัวเอง

## ข้อกำหนดเบื้องต้น

- ความรู้พื้นฐานเกี่ยวกับการเขียนโปรแกรม (ภาษาใดก็ได้)
- เข้าใจ concept พื้นฐาน เช่น variables, functions, loops
- ไม่จำเป็นต้องรู้ C มาก่อน (แต่ถ้ารู้จะช่วยได้)

## แหล่งข้อมูลเพิ่มเติม

- [Zig Official Website](https://ziglang.org)
- [Zig Documentation](https://ziglang.org/documentation/master/)
- [Zig Standard Library](https://ziglang.org/documentation/master/std/)
- [Zig GitHub](https://github.com/ziglang/zig)
- [Zig Community](https://ziggit.dev)

---

*หลักสูตรนี้ครอบคลุม Zig เวอร์ชัน 0.13.0 เป็นหลัก*
