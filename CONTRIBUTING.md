# Contributing to Mantra Programming Language

Thank you for your interest in contributing to Mantra! This guide will help you get started with contributing to our next-generation programming language.

## 🎯 Project Vision

Mantra aims to solve fundamental problems in modern programming by combining:
- **Performance + Safety** (like Rust, but easier to learn)
- **Simple Syntax** (Python-like readability with compile-time checks)
- **Strong but Flexible Typing** (gradual typing with escape hatches)
- **Better Concurrency** (built-in safe multithreading/async/GPU)
- **Unified Ecosystem** (no dependency jungle)
- **Cross-Platform First** (Web, Mobile, Embedded, Desktop)
- **AI/ML Integration** (native tensor operations)
- **Secure by Default** (memory safe, injection-proof)

## 🛠️ Getting Started

### Prerequisites

Before contributing, ensure you have:

1. **Rust Toolchain** (latest stable)
   ```bash
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   rustup component add clippy rustfmt
   ```

2. **LLVM Development Libraries** (version 15+)
   ```bash
   # Ubuntu/Debian
   sudo apt-get install llvm-dev libclang-dev
   
   # macOS
   brew install llvm
   
   # Windows
   # Download from https://releases.llvm.org/
   ```

3. **Development Tools**
   ```bash
   # Git for version control
   # CMake 3.20+ for build system
   # Your favorite editor with Rust support
   ```

### Setting Up the Development Environment

1. **Fork and Clone**
   ```bash
   git clone https://github.com/your-username/mantra.git
   cd mantra
   ```

2. **Build the Project**
   ```bash
   cargo build
   ```

3. **Run Tests**
   ```bash
   cargo test
   cargo test --release  # Run optimized tests
   ```

4. **Install Development Tools**
   ```bash
   cargo install cargo-watch  # For continuous compilation
   cargo install cargo-tarpaulin  # For code coverage
   ```

## 📁 Project Structure

Understanding the codebase structure:

```
mantra/
├── compiler/              # Core compiler implementation
│   ├── lexer/            # Tokenization and lexical analysis
│   ├── parser/           # Syntax analysis and AST generation
│   ├── semantic/         # Type checking and semantic analysis
│   ├── codegen/          # LLVM code generation
│   └── driver/           # Compiler driver and CLI
├── runtime/              # Runtime system
│   ├── memory/           # Memory management and GC
│   ├── concurrency/      # Async runtime and threading
│   └── stdlib/           # Standard library implementation
├── tools/                # Additional tools
│   ├── mantra-pkg/       # Package manager
│   ├── mantra-fmt/       # Code formatter
│   └── mantra-lsp/       # Language Server Protocol
├── tests/                # Test suites
│   ├── unit/             # Unit tests
│   ├── integration/      # Integration tests
│   ├── benchmarks/       # Performance benchmarks
│   └── examples/         # Example programs
├── docs/                 # Documentation
│   ├── spec/             # Language specification
│   ├── api/              # API documentation
│   └── tutorials/        # Learning materials
└── infra/                # Infrastructure and CI/CD
    ├── docker/           # Container configurations
    └── ci/               # Continuous integration scripts
```

## 🎯 How to Contribute

### Areas of Contribution

We welcome contributions in several areas:

#### 🔧 Core Language Development
- **Compiler Implementation** - Lexer, parser, semantic analysis
- **Code Generation** - LLVM backend optimizations
- **Type System** - Type inference, checking, and error reporting
- **Memory Management** - Ownership system and garbage collection
- **Standard Library** - Core data structures and algorithms

#### 🚀 Advanced Features
- **Concurrency Runtime** - Async/await, green threads, actors
- **GPU Computing** - CUDA/OpenCL integration
- **AI/ML Libraries** - Tensor operations, neural networks
- **Cross-Platform** - WebAssembly, mobile, embedded targets

#### 🛠️ Developer Experience
- **IDE Integration** - Language Server Protocol, VS Code extension
- **Package Manager** - Dependency resolution, registry
- **Documentation** - Tutorials, examples, API docs
- **Testing Infrastructure** - Unit tests, benchmarks, fuzz testing

#### 🌐 Community & Ecosystem
- **Examples and Tutorials** - Learning materials
- **Community Libraries** - Third-party packages
- **Language Bindings** - FFI to other languages
- **Deployment Tools** - Container images, installers

### Contribution Workflow

#### 1. Choose an Issue
- Browse [open issues](https://github.com/mantra-lang/mantra/issues)
- Look for `good-first-issue` or `help-wanted` labels
- Comment on the issue to indicate your interest

#### 2. Design Discussion
For significant features:
- Create an RFC (Request for Comments) in `docs/rfcs/`
- Discuss the design with maintainers
- Get consensus before implementation

#### 3. Implementation
- Create a feature branch: `git checkout -b feature/your-feature-name`
- Follow coding standards (see below)
- Write comprehensive tests
- Update documentation

#### 4. Testing
```bash
# Run all tests
cargo test

# Run specific test suite
cargo test --package zenith-lexer

# Run benchmarks
cargo bench

# Check code coverage
cargo tarpaulin --out Html
```

#### 5. Code Review
- Submit a pull request with clear description
- Respond to review feedback promptly
- Ensure CI passes all checks
- Update based on maintainer feedback

## 📋 Coding Standards

### Rust Code Style

Follow the official Rust style guidelines:

```bash
# Format code
cargo fmt

# Lint code
cargo clippy -- -D warnings

# Check documentation
cargo doc --no-deps
```

### Code Quality Standards

#### Documentation
```rust
/// Parses a Mantra source file into an Abstract Syntax Tree.
/// 
/// # Arguments
/// * `source` - The source code as a string
/// * `filename` - Optional filename for error reporting
/// 
/// # Returns
/// * `Ok(AST)` - Successfully parsed AST
/// * `Err(ParseError)` - Parse error with location information
/// 
/// # Examples
/// ```
/// let ast = parse_source("fn main() { print(\"Hello\") }", Some("main.mantra"))?;
/// ```
pub fn parse_source(source: &str, filename: Option<&str>) -> Result<AST, ParseError> {
    // Implementation
}
```

#### Error Handling
```rust
// Use proper error types
#[derive(Debug, thiserror::Error)]
pub enum CompilerError {
    #[error("Lexical error at {line}:{column}: {message}")]
    LexError { line: usize, column: usize, message: String },
    
    #[error("Parse error: {0}")]
    ParseError(#[from] ParseError),
    
    #[error("Type error: {0}")]
    TypeError(#[from] TypeError),
}

// Use Result types consistently
pub type CompilerResult<T> = Result<T, CompilerError>;
```

#### Testing Standards
```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_basic_arithmetic() {
        let source = "let x = 1 + 2 * 3";
        let ast = parse_source(source, None).unwrap();
        
        // Verify AST structure
        assert_matches!(ast.statements[0], 
            Statement::VarDecl { 
                name, 
                initializer: Some(Expr::Binary { .. }), 
                .. 
            } if name == "x"
        );
    }
    
    #[test]
    fn test_error_handling() {
        let invalid_source = "let x = ";
        let result = parse_source(invalid_source, None);
        
        assert!(result.is_err());
        assert_matches!(result.unwrap_err(), 
            CompilerError::ParseError(_)
        );
    }
}
```

### Commit Message Format

Use conventional commits:

```
type(scope): description

[optional body]

[optional footer]
```

Examples:
```
feat(parser): add support for async functions
fix(lexer): handle Unicode identifiers correctly
docs(readme): update installation instructions
test(semantic): add type inference test cases
```

Types:
- `feat`: New features
- `fix`: Bug fixes
- `docs`: Documentation changes
- `test`: Adding or fixing tests
- `refactor`: Code refactoring
- `perf`: Performance improvements
- `chore`: Maintenance tasks

## 🧪 Testing Guidelines

### Test Categories

#### Unit Tests
- Test individual functions/methods
- Fast execution (< 1ms per test)
- Mock external dependencies
- 100% code coverage for critical paths

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn tokenize_integer_literal() {
        let mut lexer = Lexer::new("42");
        let token = lexer.next_token().unwrap();
        
        assert_eq!(token.token_type, TokenType::Integer(42));
    }
}
```

#### Integration Tests
- Test component interactions
- Use real file I/O when appropriate
- Cover end-to-end workflows

```rust
// tests/integration/compiler_integration.rs
use std::process::Command;

#[test]
fn compile_hello_world() {
    let output = Command::new("./target/debug/mantrac")
        .arg("examples/hello_world.mantra")
        .output()
        .expect("Failed to run compiler");
    
    assert!(output.status.success());
}
```

#### Benchmark Tests
- Measure performance of critical paths
- Compare against baseline metrics
- Track performance regressions

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn benchmark_lexer(c: &mut Criterion) {
    let source = include_str!("../examples/large_file.mantra");
    
    c.bench_function("lexer_large_file", |b| {
        b.iter(|| {
            let mut lexer = Lexer::new(black_box(source));
            while !lexer.is_at_end() {
                black_box(lexer.next_token().unwrap());
            }
        })
    });
}

criterion_group!(benches, benchmark_lexer);
criterion_main!(benches);
```

### Test Data Management

- Store test files in `tests/data/`
- Use descriptive names: `tests/data/valid/async_functions.mantra`
- Include both positive and negative test cases
- Version control test data

## 📚 Documentation Standards

### API Documentation
- Document all public APIs
- Include usage examples
- Explain error conditions
- Link to relevant specifications

### User Documentation
- Write clear tutorials
- Provide complete examples
- Keep documentation up-to-date
- Test all code examples

### Internal Documentation
- Document complex algorithms
- Explain design decisions
- Link to relevant RFCs or issues
- Update when refactoring

## 🚀 Performance Guidelines

### Benchmarking
- Benchmark before and after changes
- Use realistic test data
- Measure multiple metrics (speed, memory, etc.)
- Profile to identify bottlenecks

### Optimization Priorities
1. **Correctness** - Always maintain correctness
2. **Algorithmic efficiency** - O(n) improvements
3. **Memory usage** - Reduce allocations
4. **Micro-optimizations** - When profiling shows benefit

### Performance Targets
- **Compile time**: < 1 second per 1000 lines
- **Runtime performance**: Within 20% of equivalent C++
- **Memory usage**: 50% less than equivalent Python
- **Startup time**: < 50ms for small programs

## 🤝 Community Guidelines

### Code of Conduct
- Be respectful and inclusive
- Provide constructive feedback
- Help newcomers learn
- Focus on technical merits

### Communication Channels
- **GitHub Issues**: Bug reports, feature requests
- **Discord**: Real-time chat and help
- **Forum**: Technical discussions
- **Blog**: Development updates

### Getting Help
- Read existing documentation first
- Search previous issues/discussions
- Provide minimal reproducible examples
- Be specific about your environment

## 🏆 Recognition

Contributors are recognized through:

- **Contributor list** in README
- **Changelog credits** for significant contributions
- **Community spotlights** in blog posts
- **Conference speaking opportunities**

## 📋 Release Process

### Version Numbering
We follow semantic versioning (SemVer):
- **Major**: Breaking changes (1.0.0 → 2.0.0)
- **Minor**: New features (1.0.0 → 1.1.0)
- **Patch**: Bug fixes (1.0.0 → 1.0.1)

### Release Checklist
- [ ] All tests pass
- [ ] Documentation updated
- [ ] Changelog updated
- [ ] Performance benchmarks reviewed
- [ ] Security audit completed (major releases)

## 📞 Questions?

If you have questions about contributing:

1. Check the [FAQ](docs/FAQ.md)
2. Search [existing issues](https://github.com/mantra-lang/mantra/issues)
3. Ask on [Discord](https://discord.gg/mantra-lang)
4. Create a new issue with the `question` label

Thank you for helping build the future of programming languages! 🚀
