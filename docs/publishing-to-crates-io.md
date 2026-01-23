# 发布 Crate 到 crates.io 注意事项

本文档总结了将 Rust crate 发布到 crates.io 的过程中需要注意的关键事项，基于 rCore-Tutorial-in-single-workspace 项目的实际发布经验整理。

## 目录

1. [Crate 命名](#1-crate-命名)
2. [Cargo.toml 必填信息](#2-cargotoml-必填信息)
3. [依赖引用格式](#3-依赖引用格式)
4. [Workspace 配置优化](#4-workspace-配置优化)
5. [代码格式化与 Lint 检查](#5-代码格式化与-lint-检查)
6. [文档要求](#6-文档要求)
7. [跨平台兼容性](#7-跨平台兼容性)
8. [docs.rs 构建配置](#8-docsrs-构建配置)
9. [Build Script 注意事项](#9-build-script-注意事项)
10. [发布前检查](#10-发布前检查)
11. [发布顺序](#11-发布顺序)

---

## 1. Crate 命名

### 问题

crates.io 上的 crate 名称是全局唯一的，常见名称如 `console`、`sync`、`signal` 等很可能已被占用。

### 解决方案

为所有公开发布的 crate 添加统一前缀，例如：

```
console      ->  tg-console
easy-fs      ->  tg-easy-fs
kernel-alloc ->  tg-kernel-alloc
signal       ->  tg-signal
sync         ->  tg-sync
syscall      ->  tg-syscall
...
```

**注意**：修改 crate 名称后，需要同步更新所有引用该 crate 的地方：
- 其他 crate 的 `Cargo.toml` 依赖
- Rust 代码中的 `use` 语句（`tg-console` → `tg_console`）

---

## 2. Cargo.toml 必填信息

发布到 crates.io 需要在 `Cargo.toml` 中包含以下字段：

### 必填字段

```toml
[package]
name = "tg-syscall"                        # crate 名称
version = "0.1.0"                          # 语义化版本号
edition = "2021"                           # Rust edition
description = "System call definitions..." # 简短描述（必填）
license = "MIT OR Apache-2.0"              # 许可证（必填）
```

### 强烈建议字段

```toml
[package]
authors = ["Author <email@example.com>"]
repository = "https://github.com/org/repo"   # 源码仓库地址
homepage = "https://github.com/org/repo"     # 项目主页
documentation = "https://docs.rs/tg-syscall" # 文档地址
readme = "README.md"                         # README 文件
keywords = ["rcore", "syscall", "no-std", "riscv"]  # 最多 5 个关键词
categories = ["no-std", "embedded"]          # 分类
```

### 完整示例

```toml
[package]
name = "tg-syscall"
description = "System call definitions and interfaces for rCore tutorial OS."
version = "0.1.0-preview.1"
edition = "2021"
authors = ["YdrMaster <ydrml@hotmail.com>"]
repository = "https://github.com/rcore-os/rCore-Tutorial-in-single-workspace"
homepage = "https://github.com/rcore-os/rCore-Tutorial-in-single-workspace/tree/tg-preview"
documentation = "https://docs.rs/tg-syscall"
license = "MIT OR Apache-2.0"
readme = "README.md"
keywords = ["rcore", "syscall", "no-std", "riscv"]
categories = ["no-std", "embedded"]
```

---

## 3. 依赖引用格式

### 路径依赖必须同时指定版本

当 workspace 内部 crate 相互依赖时，如果这些 crate 都要发布到 crates.io，**必须同时指定 `path` 和 `version`**：

```toml
# ❌ 错误：只有 path，cargo publish 会失败
[dependencies]
tg-task-manage = { path = "../tg-task-manage" }

# ✅ 正确：同时指定 path 和 version
[dependencies]
tg-task-manage = { path = "../tg-task-manage", version = "0.1" }
```

**原理**：
- 本地开发时 Cargo 使用 `path` 指定的本地代码
- 发布时 Cargo 会忽略 `path`，使用 `version` 从 crates.io 获取依赖

### 带 features 的依赖

```toml
tg-task-manage = { path = "../tg-task-manage", version = "0.1", features = ["thread"] }
```

---

## 4. Workspace 配置优化

使用 `workspace.package` 和 `workspace.dependencies` 统一管理多个 crate 的配置。

### 根 Cargo.toml 配置

```toml
[workspace]
members = ["tg-console", "tg-syscall", "tg-signal", ...]
resolver = "2"

# 统一的包元数据，子 crate 可通过 `xxx.workspace = true` 继承
[workspace.package]
version = "0.1.0-preview.1"
edition = "2021"
repository = "https://github.com/rcore-os/rCore-Tutorial-in-single-workspace"
license = "MIT OR Apache-2.0"
keywords = ["rcore", "os", "riscv", "tutorial"]
categories = ["no-std", "embedded"]

# 统一的依赖版本管理
[workspace.dependencies]
# 外部依赖
spin = "0.9"
bitflags = "1.2"
log = "0.4"

# 内部 crate 依赖
tg-console = { path = "tg-console", version = "0.1.0-preview.1" }
tg-syscall = { path = "tg-syscall", version = "0.1.0-preview.1" }
tg-signal = { path = "tg-signal", version = "0.1.0-preview.1" }
```

### 子 Crate 使用 workspace 配置

```toml
[package]
name = "tg-signal-impl"
version.workspace = true
edition.workspace = true
license.workspace = true
# ... 其他继承的字段

[dependencies]
spin.workspace = true
tg-signal.workspace = true
```

---

## 5. 代码格式化与 Lint 检查

发布前务必运行格式化和 lint 检查：

```bash
# 格式化代码
cargo fmt --all

# 运行 clippy 检查
cargo clippy --all-targets --all-features -- -D warnings
```

### 常见 clippy 警告修复

1. **缺少 `is_empty()` 方法**
   ```rust
   // 如果实现了 len()，clippy 建议同时实现 is_empty()
   pub fn is_empty(&self) -> bool {
       self.len() == 0
   }
   ```

2. **代码格式问题**
   - 运行 `cargo fmt` 自动修复

3. **未使用的导入/变量**
   - 删除或使用 `#[allow(unused)]`（仅在确实需要时）

---

## 6. 文档要求

### README.md

为每个要发布的 crate 准备完善的 README：

```markdown
# crate-name

[![Crates.io](https://img.shields.io/crates/v/crate-name.svg)](https://crates.io/crates/crate-name)
[![Documentation](https://docs.rs/crate-name/badge.svg)](https://docs.rs/crate-name)
[![License](https://img.shields.io/crates/l/crate-name.svg)](LICENSE)

简短描述

## Overview
详细介绍

## Features
功能列表

## Usage
使用示例

## License
许可证信息
```

### Crate 级文档

在 `lib.rs` 顶部添加 crate 级文档：

```rust
//! # tg-signal
//!
//! Signal handling infrastructure for the rCore tutorial operating system.
//!
//! This crate provides ...
```

### SAFETY 注释

**所有 `unsafe` 代码块必须添加 `SAFETY` 注释**，说明为何该代码是安全的：

```rust
/// 初始化内存分配。
///
/// # Safety
///
/// 调用者必须确保：
/// - `region` 内存块与已经转移到分配器的内存块都不重叠
/// - `region` 未被其他对象引用
pub unsafe fn transfer(region: &'static mut [u8]) {
    let ptr = NonNull::new(region.as_mut_ptr()).unwrap();
    // SAFETY: 由调用者保证内存块有效且不重叠
    HEAP.transfer(ptr, region.len());
}
```

### 启用文档 lint

建议在发布的 crate 中启用严格的文档检查：

```rust
#![deny(warnings, missing_docs)]
```

---

## 7. 跨平台兼容性

### 问题

crates.io 和 docs.rs 会在标准 x86_64 环境下构建和测试 crate。包含特定架构代码（如 RISC-V 内联汇编）的 crate 会编译失败。

### 解决方案

使用 `#[cfg(target_arch)]` 条件编译：

```rust
/// RISC-V 64 位架构的实现
#[cfg(target_arch = "riscv64")]
pub mod native {
    use core::arch::asm;
    
    #[inline(always)]
    pub unsafe fn syscall0(id: SyscallId) -> isize {
        let ret: isize;
        asm!(
            "ecall",
            in("a7") id as usize,
            lateout("a0") ret,
        );
        ret
    }
}

/// 非 riscv64 架构的 stub 实现
#[cfg(not(target_arch = "riscv64"))]
pub mod native {
    use crate::SyscallId;
    
    #[inline(always)]
    pub unsafe fn syscall0(_id: SyscallId) -> isize {
        unimplemented!("syscall is only supported on riscv64")
    }
}
```

### 宏中的条件编译

```rust
#[macro_export]
macro_rules! boot0 {
    ($entry:ident; $stack_size:expr) => {
        #[cfg(target_arch = "riscv64")]
        #[naked]
        #[no_mangle]
        #[link_section = ".text.entry"]
        unsafe extern "C" fn _start() -> ! {
            core::arch::asm!(
                "la sp, {boot_stack_top}",
                "j  {main}",
                boot_stack_top = sym $stack,
                main = sym $entry,
                options(noreturn)
            )
        }
        
        // 非 riscv64 架构下不生成 _start 函数
    };
}
```

---

## 8. docs.rs 构建配置

### 指定构建目标

对于特定架构的 crate，需要告诉 docs.rs 使用正确的 target：

```toml
[package.metadata.docs.rs]
targets = ["riscv64gc-unknown-none-elf"]
```

这样 docs.rs 会使用指定的 target 来构建文档，避免因架构不兼容导致文档生成失败。

### 启用特定 features

```toml
[package.metadata.docs.rs]
targets = ["riscv64gc-unknown-none-elf"]
all-features = true
# 或者指定特定 features
features = ["kernel", "user"]
```

---

## 9. Build Script 注意事项

### 问题

如果 `build.rs` 在 `src/` 目录下生成代码文件，`cargo publish` 会失败，因为 cargo 不允许 build script 修改源码目录。

### 解决方案

将生成的文件输出到 `OUT_DIR`：

```rust
// build.rs
fn main() {
    use std::{env, fs::File, io::Write, path::PathBuf};
    
    println!("cargo:rerun-if-changed=build.rs");
    println!("cargo:rerun-if-changed=src/syscall.h.in");
    
    // ✅ 正确：输出到 OUT_DIR
    let out_dir = PathBuf::from(env::var("OUT_DIR").unwrap());
    let mut fout = File::create(out_dir.join("syscalls.rs")).unwrap();
    
    // 生成代码...
}
```

在 lib.rs 中使用 `include!` 引入生成的文件：

```rust
// src/lib.rs

// ❌ 错误：直接引用 src 目录下的文件
// mod syscalls;

// ✅ 正确：从 OUT_DIR 引入
include!(concat!(env!("OUT_DIR"), "/syscalls.rs"));
```

---

## 10. 发布前检查

### 使用 dry-run 测试

在正式发布前，务必运行 dry-run 检查：

```bash
# 检查单个 crate
cargo publish --dry-run -p tg-console

# 检查所有要发布的 crate
for crate in tg-console tg-linker tg-syscall; do
    echo "Checking $crate..."
    cargo publish --dry-run -p $crate
done
```

### 常见 dry-run 错误

1. **缺少必填字段**
   ```
   error: missing field `description`
   ```

2. **路径依赖没有版本号**
   ```
   error: dependency 'tg-xxx' has not been uploaded to crates.io
   ```

3. **build script 修改 src 目录**
   ```
   error: cannot update files within source directory
   ```

4. **依赖未发布**
   ```
   error: dependency 'tg-xxx' not found in registry
   ```

### 检查清单

- [ ] `cargo fmt --check` 通过
- [ ] `cargo clippy` 无警告
- [ ] `cargo doc --no-deps` 成功生成文档
- [ ] `cargo publish --dry-run` 成功
- [ ] README.md 完整且正确
- [ ] SAFETY 注释完整
- [ ] LICENSE 文件存在
- [ ] 依赖的 crate 已发布（或同时发布）

---

## 11. 发布顺序

### 依赖分析

当多个 crate 相互依赖时，必须按照依赖顺序从底层到上层发布：

```
层级关系示例：
Layer 0 (无内部依赖):
  - tg-console
  - tg-linker  
  - tg-kernel-alloc
  - tg-signal-defs

Layer 1 (依赖 Layer 0):
  - tg-syscall      -> tg-signal-defs
  - tg-kernel-context
  - tg-signal       -> tg-kernel-context, tg-signal-defs

Layer 2 (依赖 Layer 1):
  - tg-kernel-vm    -> tg-kernel-context
  - tg-signal-impl  -> tg-kernel-context, tg-signal
  - tg-task-manage

Layer 3 (依赖 Layer 2):
  - tg-sync         -> tg-task-manage
  - tg-easy-fs
```

### 发布脚本示例

```bash
#!/bin/bash
set -e

CRATES=(
    # Layer 0
    "tg-console"
    "tg-linker"
    "tg-kernel-alloc"
    "tg-signal-defs"
    
    # Layer 1
    "tg-syscall"
    "tg-kernel-context"
    "tg-signal"
    
    # Layer 2
    "tg-kernel-vm"
    "tg-signal-impl"
    "tg-task-manage"
    
    # Layer 3
    "tg-sync"
    "tg-easy-fs"
)

for crate in "${CRATES[@]}"; do
    echo "Publishing $crate..."
    cargo publish -p "$crate"
    
    # 等待 crates.io 索引更新
    echo "Waiting for index to update..."
    sleep 30
done

echo "All crates published successfully!"
```

### 注意事项

1. **等待索引更新**：每发布一个 crate 后，需要等待 crates.io 更新索引（通常 10-30 秒）
2. **版本一致性**：确保所有相互依赖的 crate 使用一致的版本号
3. **发布失败处理**：如果中途失败，需要从失败的 crate 重新开始

---

## 总结

发布 Rust crate 到 crates.io 需要注意以下核心要点：

1. ✅ 选择唯一的 crate 名称（添加前缀）
2. ✅ 完善 Cargo.toml 元信息
3. ✅ 路径依赖必须同时指定版本
4. ✅ 运行 `cargo fmt` 和 `cargo clippy`
5. ✅ 添加完整的文档和 SAFETY 注释
6. ✅ 处理跨平台兼容性（`#[cfg(target_arch)]`）
7. ✅ 配置 docs.rs 构建目标
8. ✅ build script 生成文件到 OUT_DIR
9. ✅ 使用 `cargo publish --dry-run` 预检查
10. ✅ 按依赖顺序发布

遵循以上步骤，可以顺利将 crate 发布到 crates.io！
