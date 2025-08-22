# Implementation Guide: Building the Mantra Compiler

This guide provides detailed technical instructions for implementing each phase of the Mantra programming language development.

## Phase 1: Foundation Implementation

### 1.1 Setting Up the Development Environment

#### Prerequisites Installation
```bash
# Install Rust toolchain (for compiler implementation)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup component add clippy rustfmt

# Install LLVM development libraries
# Ubuntu/Debian:
sudo apt-get install llvm-dev libclang-dev

# macOS:
brew install llvm

# Windows:
# Download LLVM pre-built binaries from https://releases.llvm.org/
```

#### Project Structure Setup
```
mantra/
├── compiler/
│   ├── lexer/          # Lexical analysis
│   ├── parser/         # Syntax analysis
│   ├── semantic/       # Semantic analysis
│   ├── codegen/        # Code generation
│   └── driver/         # Compiler driver
├── runtime/
│   ├── memory/         # Memory management
│   ├── concurrency/    # Async runtime
│   └── stdlib/         # Standard library
├── tools/
│   ├── mantra-pkg/     # Package manager
│   └── mantra-fmt/     # Code formatter
├── tests/
│   ├── unit/           # Unit tests
│   ├── integration/    # Integration tests
│   └── benchmarks/     # Performance benchmarks
└── docs/               # Documentation
```

### 1.2 Lexer Implementation

#### Token Definition
```rust
// compiler/lexer/token.rs
#[derive(Debug, Clone, PartialEq)]
pub enum TokenType {
    // Literals
    Integer(i64),
    Float(f64),
    String(String),
    Character(char),
    Boolean(bool),
    
    // Identifiers and Keywords
    Identifier(String),
    Keyword(Keyword),
    
    // Operators
    Plus, Minus, Multiply, Divide, Modulo,
    Equal, NotEqual, Less, Greater, LessEqual, GreaterEqual,
    And, Or, Not,
    Assign, PlusAssign, MinusAssign,
    
    // Delimiters
    LeftParen, RightParen,
    LeftBrace, RightBrace,
    LeftBracket, RightBracket,
    Comma, Semicolon, Colon, Dot,
    
    // Special
    Arrow, FatArrow, Question, DoubleQuestion,
    Newline, Whitespace, Comment,
    EndOfFile,
}

#[derive(Debug, Clone, PartialEq)]
pub enum Keyword {
    Fn, Let, Mut, Const, If, Else, While, For, Loop,
    Return, Break, Continue, Match, Async, Await,
    Struct, Enum, Trait, Impl, Mod, Use, Pub,
    True, False, Null,
}
```

#### Lexer Core Implementation
```rust
// compiler/lexer/lexer.rs
pub struct Lexer {
    input: Vec<char>,
    position: usize,
    line: usize,
    column: usize,
}

impl Lexer {
    pub fn new(input: &str) -> Self {
        Self {
            input: input.chars().collect(),
            position: 0,
            line: 1,
            column: 1,
        }
    }
    
    pub fn next_token(&mut self) -> Result<Token, LexError> {
        self.skip_whitespace();
        
        if self.is_at_end() {
            return Ok(Token::new(TokenType::EndOfFile, self.line, self.column));
        }
        
        let ch = self.current_char();
        
        match ch {
            '(' => self.single_char_token(TokenType::LeftParen),
            ')' => self.single_char_token(TokenType::RightParen),
            '{' => self.single_char_token(TokenType::LeftBrace),
            '}' => self.single_char_token(TokenType::RightBrace),
            '[' => self.single_char_token(TokenType::LeftBracket),
            ']' => self.single_char_token(TokenType::RightBracket),
            ',' => self.single_char_token(TokenType::Comma),
            ';' => self.single_char_token(TokenType::Semicolon),
            ':' => self.single_char_token(TokenType::Colon),
            '.' => self.single_char_token(TokenType::Dot),
            
            '+' => self.handle_plus(),
            '-' => self.handle_minus(),
            '*' => self.single_char_token(TokenType::Multiply),
            '/' => self.handle_slash(),
            '%' => self.single_char_token(TokenType::Modulo),
            
            '=' => self.handle_equal(),
            '!' => self.handle_bang(),
            '<' => self.handle_less(),
            '>' => self.handle_greater(),
            
            '"' => self.string_literal(),
            '\'' => self.character_literal(),
            
            '0'..='9' => self.number_literal(),
            'a'..='z' | 'A'..='Z' | '_' => self.identifier_or_keyword(),
            
            _ => Err(LexError::UnexpectedCharacter(ch, self.line, self.column)),
        }
    }
    
    fn identifier_or_keyword(&mut self) -> Result<Token, LexError> {
        let start_pos = self.position;
        
        while self.is_alphanumeric_or_underscore() {
            self.advance();
        }
        
        let text: String = self.input[start_pos..self.position].iter().collect();
        let token_type = match text.as_str() {
            "fn" => TokenType::Keyword(Keyword::Fn),
            "let" => TokenType::Keyword(Keyword::Let),
            "mut" => TokenType::Keyword(Keyword::Mut),
            "if" => TokenType::Keyword(Keyword::If),
            "else" => TokenType::Keyword(Keyword::Else),
            "true" => TokenType::Boolean(true),
            "false" => TokenType::Boolean(false),
            "null" => TokenType::Keyword(Keyword::Null),
            // ... other keywords
            _ => TokenType::Identifier(text),
        };
        
        Ok(Token::new(token_type, self.line, self.column))
    }
    
    fn number_literal(&mut self) -> Result<Token, LexError> {
        let start_pos = self.position;
        
        // Handle integer part
        while self.is_digit() {
            self.advance();
        }
        
        // Handle decimal part
        if self.current_char() == '.' && self.peek_char().map_or(false, |c| c.is_ascii_digit()) {
            self.advance(); // consume '.'
            while self.is_digit() {
                self.advance();
            }
            
            let text: String = self.input[start_pos..self.position].iter().collect();
            let value: f64 = text.parse().map_err(|_| LexError::InvalidNumber)?;
            Ok(Token::new(TokenType::Float(value), self.line, self.column))
        } else {
            let text: String = self.input[start_pos..self.position].iter().collect();
            let value: i64 = text.parse().map_err(|_| LexError::InvalidNumber)?;
            Ok(Token::new(TokenType::Integer(value), self.line, self.column))
        }
    }
}
```

### 1.3 Parser Implementation

#### Abstract Syntax Tree (AST) Definition
```rust
// compiler/parser/ast.rs
#[derive(Debug, Clone)]
pub enum Expr {
    // Literals
    Integer(i64),
    Float(f64),
    String(String),
    Boolean(bool),
    Null,
    
    // Identifiers
    Identifier(String),
    
    // Binary operations
    Binary {
        left: Box<Expr>,
        operator: BinaryOp,
        right: Box<Expr>,
    },
    
    // Unary operations
    Unary {
        operator: UnaryOp,
        operand: Box<Expr>,
    },
    
    // Function calls
    Call {
        callee: Box<Expr>,
        arguments: Vec<Expr>,
    },
    
    // Field access
    FieldAccess {
        object: Box<Expr>,
        field: String,
    },
    
    // Array/Index access
    Index {
        object: Box<Expr>,
        index: Box<Expr>,
    },
}

#[derive(Debug, Clone)]
pub enum Stmt {
    // Expression statement
    Expression(Expr),
    
    // Variable declaration
    VarDecl {
        name: String,
        type_annotation: Option<Type>,
        initializer: Option<Expr>,
        is_mutable: bool,
    },
    
    // Function declaration
    FnDecl {
        name: String,
        parameters: Vec<Parameter>,
        return_type: Option<Type>,
        body: Block,
        is_async: bool,
    },
    
    // If statement
    If {
        condition: Expr,
        then_branch: Block,
        else_branch: Option<Block>,
    },
    
    // While loop
    While {
        condition: Expr,
        body: Block,
    },
    
    // Return statement
    Return(Option<Expr>),
    
    // Block
    Block(Block),
}

#[derive(Debug, Clone)]
pub struct Block {
    pub statements: Vec<Stmt>,
}

#[derive(Debug, Clone)]
pub struct Parameter {
    pub name: String,
    pub type_annotation: Type,
}
```

#### Recursive Descent Parser
```rust
// compiler/parser/parser.rs
pub struct Parser {
    tokens: Vec<Token>,
    current: usize,
}

impl Parser {
    pub fn new(tokens: Vec<Token>) -> Self {
        Self { tokens, current: 0 }
    }
    
    pub fn parse(&mut self) -> Result<Vec<Stmt>, ParseError> {
        let mut statements = Vec::new();
        
        while !self.is_at_end() {
            statements.push(self.declaration()?);
        }
        
        Ok(statements)
    }
    
    fn declaration(&mut self) -> Result<Stmt, ParseError> {
        if self.match_token(&TokenType::Keyword(Keyword::Fn)) {
            self.function_declaration()
        } else if self.match_token(&TokenType::Keyword(Keyword::Let)) {
            self.variable_declaration(false)
        } else if self.match_token(&TokenType::Keyword(Keyword::Mut)) {
            self.variable_declaration(true)
        } else {
            self.statement()
        }
    }
    
    fn function_declaration(&mut self) -> Result<Stmt, ParseError> {
        let name = self.consume_identifier("Expected function name")?;
        
        self.consume(&TokenType::LeftParen, "Expected '(' after function name")?;
        
        let mut parameters = Vec::new();
        if !self.check(&TokenType::RightParen) {
            loop {
                let param_name = self.consume_identifier("Expected parameter name")?;
                self.consume(&TokenType::Colon, "Expected ':' after parameter name")?;
                let param_type = self.parse_type()?;
                
                parameters.push(Parameter {
                    name: param_name,
                    type_annotation: param_type,
                });
                
                if !self.match_token(&TokenType::Comma) {
                    break;
                }
            }
        }
        
        self.consume(&TokenType::RightParen, "Expected ')' after parameters")?;
        
        let return_type = if self.match_token(&TokenType::Arrow) {
            Some(self.parse_type()?)
        } else {
            None
        };
        
        let body = self.block_statement()?;
        
        Ok(Stmt::FnDecl {
            name,
            parameters,
            return_type,
            body,
            is_async: false, // TODO: Handle async keyword
        })
    }
    
    fn expression(&mut self) -> Result<Expr, ParseError> {
        self.assignment()
    }
    
    fn assignment(&mut self) -> Result<Expr, ParseError> {
        let expr = self.logical_or()?;
        
        if self.match_token(&TokenType::Assign) {
            let value = self.assignment()?;
            // TODO: Handle assignment
        }
        
        Ok(expr)
    }
    
    fn logical_or(&mut self) -> Result<Expr, ParseError> {
        let mut expr = self.logical_and()?;
        
        while self.match_token(&TokenType::Or) {
            let operator = BinaryOp::Or;
            let right = self.logical_and()?;
            expr = Expr::Binary {
                left: Box::new(expr),
                operator,
                right: Box::new(right),
            };
        }
        
        Ok(expr)
    }
    
    fn primary(&mut self) -> Result<Expr, ParseError> {
        if let Some(token) = self.peek() {
            match &token.token_type {
                TokenType::Integer(value) => {
                    self.advance();
                    Ok(Expr::Integer(*value))
                }
                TokenType::Float(value) => {
                    self.advance();
                    Ok(Expr::Float(*value))
                }
                TokenType::String(value) => {
                    self.advance();
                    Ok(Expr::String(value.clone()))
                }
                TokenType::Boolean(value) => {
                    self.advance();
                    Ok(Expr::Boolean(*value))
                }
                TokenType::Identifier(name) => {
                    self.advance();
                    Ok(Expr::Identifier(name.clone()))
                }
                TokenType::LeftParen => {
                    self.advance(); // consume '('
                    let expr = self.expression()?;
                    self.consume(&TokenType::RightParen, "Expected ')' after expression")?;
                    Ok(expr)
                }
                _ => Err(ParseError::UnexpectedToken(token.clone())),
            }
        } else {
            Err(ParseError::UnexpectedEndOfFile)
        }
    }
}
```

### 1.4 Type System Foundation

#### Type Representation
```rust
// compiler/semantic/types.rs
#[derive(Debug, Clone, PartialEq)]
pub enum Type {
    // Primitive types
    Integer(IntegerType),
    Float(FloatType),
    Boolean,
    Character,
    String,
    Unit, // equivalent to void
    
    // Composite types
    Array {
        element_type: Box<Type>,
        size: Option<usize>, // None for dynamic arrays
    },
    Tuple(Vec<Type>),
    Function {
        parameters: Vec<Type>,
        return_type: Box<Type>,
    },
    
    // User-defined types
    Struct {
        name: String,
        fields: Vec<(String, Type)>,
    },
    Enum {
        name: String,
        variants: Vec<EnumVariant>,
    },
    
    // Generic types
    Generic {
        name: String,
        type_params: Vec<Type>,
    },
    TypeParameter(String),
    
    // Special types
    Optional(Box<Type>),
    Result {
        ok_type: Box<Type>,
        err_type: Box<Type>,
    },
    
    // Inference placeholder
    Unknown,
}

#[derive(Debug, Clone, PartialEq)]
pub enum IntegerType {
    I8, I16, I32, I64, I128,
    U8, U16, U32, U64, U128,
    Int,  // Platform-dependent
    UInt, // Platform-dependent
}

#[derive(Debug, Clone, PartialEq)]
pub enum FloatType {
    F32, F64,
    Float, // Default (F64)
}
```

#### Type Checker Implementation
```rust
// compiler/semantic/type_checker.rs
pub struct TypeChecker {
    symbol_table: SymbolTable,
    current_scope: ScopeId,
    type_env: TypeEnvironment,
}

impl TypeChecker {
    pub fn new() -> Self {
        Self {
            symbol_table: SymbolTable::new(),
            current_scope: ScopeId::global(),
            type_env: TypeEnvironment::new(),
        }
    }
    
    pub fn check_program(&mut self, program: &[Stmt]) -> Result<(), TypeError> {
        for stmt in program {
            self.check_statement(stmt)?;
        }
        Ok(())
    }
    
    fn check_statement(&mut self, stmt: &Stmt) -> Result<Type, TypeError> {
        match stmt {
            Stmt::VarDecl { name, type_annotation, initializer, .. } => {
                let var_type = if let Some(init) = initializer {
                    let inferred_type = self.check_expression(init)?;
                    
                    if let Some(annotation) = type_annotation {
                        if !self.types_compatible(&inferred_type, annotation) {
                            return Err(TypeError::TypeMismatch {
                                expected: annotation.clone(),
                                actual: inferred_type,
                            });
                        }
                        annotation.clone()
                    } else {
                        inferred_type
                    }
                } else if let Some(annotation) = type_annotation {
                    annotation.clone()
                } else {
                    return Err(TypeError::CannotInferType(name.clone()));
                };
                
                self.symbol_table.insert_variable(name.clone(), var_type.clone())?;
                Ok(var_type)
            }
            
            Stmt::FnDecl { name, parameters, return_type, body, .. } => {
                // Create function type
                let param_types: Vec<Type> = parameters.iter()
                    .map(|p| p.type_annotation.clone())
                    .collect();
                
                let ret_type = return_type.clone().unwrap_or(Type::Unit);
                
                let fn_type = Type::Function {
                    parameters: param_types,
                    return_type: Box::new(ret_type.clone()),
                };
                
                self.symbol_table.insert_function(name.clone(), fn_type)?;
                
                // Check function body
                self.enter_scope();
                
                // Add parameters to scope
                for param in parameters {
                    self.symbol_table.insert_variable(
                        param.name.clone(), 
                        param.type_annotation.clone()
                    )?;
                }
                
                let body_type = self.check_block(body)?;
                
                // Verify return type
                if !self.types_compatible(&body_type, &ret_type) {
                    return Err(TypeError::ReturnTypeMismatch {
                        expected: ret_type,
                        actual: body_type,
                    });
                }
                
                self.exit_scope();
                Ok(Type::Unit)
            }
            
            _ => todo!("Implement other statement types"),
        }
    }
    
    fn check_expression(&mut self, expr: &Expr) -> Result<Type, TypeError> {
        match expr {
            Expr::Integer(_) => Ok(Type::Integer(IntegerType::Int)),
            Expr::Float(_) => Ok(Type::Float(FloatType::Float)),
            Expr::String(_) => Ok(Type::String),
            Expr::Boolean(_) => Ok(Type::Boolean),
            Expr::Null => Ok(Type::Optional(Box::new(Type::Unknown))),
            
            Expr::Identifier(name) => {
                self.symbol_table.lookup_variable(name)
                    .ok_or_else(|| TypeError::UndefinedVariable(name.clone()))
            }
            
            Expr::Binary { left, operator, right } => {
                let left_type = self.check_expression(left)?;
                let right_type = self.check_expression(right)?;
                
                self.check_binary_operation(&left_type, operator, &right_type)
            }
            
            Expr::Call { callee, arguments } => {
                let callee_type = self.check_expression(callee)?;
                
                if let Type::Function { parameters, return_type } = callee_type {
                    if arguments.len() != parameters.len() {
                        return Err(TypeError::ArgumentCountMismatch {
                            expected: parameters.len(),
                            actual: arguments.len(),
                        });
                    }
                    
                    for (arg, param_type) in arguments.iter().zip(parameters.iter()) {
                        let arg_type = self.check_expression(arg)?;
                        if !self.types_compatible(&arg_type, param_type) {
                            return Err(TypeError::ArgumentTypeMismatch {
                                expected: param_type.clone(),
                                actual: arg_type,
                            });
                        }
                    }
                    
                    Ok(*return_type)
                } else {
                    Err(TypeError::NotCallable(callee_type))
                }
            }
            
            _ => todo!("Implement other expression types"),
        }
    }
    
    fn types_compatible(&self, actual: &Type, expected: &Type) -> bool {
        match (actual, expected) {
            (Type::Unknown, _) | (_, Type::Unknown) => true,
            (a, b) if a == b => true,
            
            // Integer type compatibility
            (Type::Integer(actual_int), Type::Integer(expected_int)) => {
                self.integer_types_compatible(actual_int, expected_int)
            }
            
            // Optional type handling
            (t, Type::Optional(inner)) => self.types_compatible(t, inner),
            (Type::Optional(inner), t) => self.types_compatible(inner, t),
            
            _ => false,
        }
    }
}
```

## Phase 2: Advanced Implementation Details

### 2.1 LLVM Code Generation

```rust
// compiler/codegen/llvm_gen.rs
use llvm_sys::*;

pub struct LLVMCodeGenerator {
    context: LLVMContextRef,
    module: LLVMModuleRef,
    builder: LLVMBuilderRef,
    values: HashMap<String, LLVMValueRef>,
}

impl LLVMCodeGenerator {
    pub fn new(module_name: &str) -> Self {
        unsafe {
            let context = LLVMContextCreate();
            let module = LLVMModuleCreateWithNameInContext(
                module_name.as_ptr() as *const i8,
                context
            );
            let builder = LLVMCreateBuilderInContext(context);
            
            Self {
                context,
                module,
                builder,
                values: HashMap::new(),
            }
        }
    }
    
    pub fn generate_function(&mut self, func: &FunctionDecl) -> Result<LLVMValueRef, CodegenError> {
        // Generate function signature
        let param_types: Vec<LLVMTypeRef> = func.parameters.iter()
            .map(|p| self.llvm_type(&p.type_annotation))
            .collect::<Result<Vec<_>, _>>()?;
        
        let return_type = if let Some(ret_type) = &func.return_type {
            self.llvm_type(ret_type)?
        } else {
            unsafe { LLVMVoidTypeInContext(self.context) }
        };
        
        let function_type = unsafe {
            LLVMFunctionType(
                return_type,
                param_types.as_ptr() as *mut LLVMTypeRef,
                param_types.len() as u32,
                0 // not variadic
            )
        };
        
        let function = unsafe {
            LLVMAddFunction(
                self.module,
                func.name.as_ptr() as *const i8,
                function_type
            )
        };
        
        // Create entry basic block
        let entry_block = unsafe {
            LLVMAppendBasicBlockInContext(
                self.context,
                function,
                "entry\0".as_ptr() as *const i8
            )
        };
        
        unsafe {
            LLVMPositionBuilderAtEnd(self.builder, entry_block);
        }
        
        // Generate function body
        self.generate_block(&func.body)?;
        
        Ok(function)
    }
    
    fn llvm_type(&self, ty: &Type) -> Result<LLVMTypeRef, CodegenError> {
        unsafe {
            match ty {
                Type::Integer(IntegerType::I32) => Ok(LLVMInt32TypeInContext(self.context)),
                Type::Integer(IntegerType::I64) => Ok(LLVMInt64TypeInContext(self.context)),
                Type::Float(FloatType::F64) => Ok(LLVMDoubleTypeInContext(self.context)),
                Type::Boolean => Ok(LLVMInt1TypeInContext(self.context)),
                Type::Unit => Ok(LLVMVoidTypeInContext(self.context)),
                
                Type::Array { element_type, size } => {
                    let elem_type = self.llvm_type(element_type)?;
                    if let Some(size) = size {
                        Ok(LLVMArrayType(elem_type, *size as u32))
                    } else {
                        // Dynamic array - represented as pointer
                        Ok(LLVMPointerType(elem_type, 0))
                    }
                }
                
                _ => Err(CodegenError::UnsupportedType(ty.clone())),
            }
        }
    }
}
```

### 2.2 Memory Management Implementation

```rust
// runtime/memory/allocator.rs
use std::alloc::{GlobalAlloc, Layout};
use std::sync::atomic::{AtomicUsize, Ordering};

pub struct MantraAllocator {
    allocated: AtomicUsize,
    peak_allocated: AtomicUsize,
}

impl MantraAllocator {
    pub const fn new() -> Self {
        Self {
            allocated: AtomicUsize::new(0),
            peak_allocated: AtomicUsize::new(0),
        }
    }
    
    pub fn allocated_bytes(&self) -> usize {
        self.allocated.load(Ordering::SeqCst)
    }
    
    pub fn peak_allocated_bytes(&self) -> usize {
        self.peak_allocated.load(Ordering::SeqCst)
    }
}

unsafe impl GlobalAlloc for MantraAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let ptr = std::alloc::System.alloc(layout);
        if !ptr.is_null() {
            let current = self.allocated.fetch_add(layout.size(), Ordering::SeqCst);
            let new_total = current + layout.size();
            
            // Update peak if necessary
            self.peak_allocated.fetch_max(new_total, Ordering::SeqCst);
        }
        ptr
    }
    
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        std::alloc::System.dealloc(ptr, layout);
        self.allocated.fetch_sub(layout.size(), Ordering::SeqCst);
    }
}

// Ownership and borrowing system
pub struct Owned<T> {
    value: T,
    _marker: std::marker::PhantomData<T>,
}

impl<T> Owned<T> {
    pub fn new(value: T) -> Self {
        Self {
            value,
            _marker: std::marker::PhantomData,
        }
    }
    
    pub fn borrow(&self) -> Borrowed<T> {
        Borrowed::new(&self.value)
    }
    
    pub fn borrow_mut(&mut self) -> BorrowedMut<T> {
        BorrowedMut::new(&mut self.value)
    }
}

pub struct Borrowed<T> {
    value: *const T,
    _marker: std::marker::PhantomData<&'static T>,
}

impl<T> Borrowed<T> {
    fn new(value: &T) -> Self {
        Self {
            value: value as *const T,
            _marker: std::marker::PhantomData,
        }
    }
}

unsafe impl<T: Send> Send for Borrowed<T> {}
unsafe impl<T: Sync> Sync for Borrowed<T> {}
```

This implementation guide provides the concrete technical foundation for building the Mantra programming language. Each phase builds upon the previous one, creating a robust and maintainable compiler architecture that can achieve the performance, safety, and usability goals outlined in the roadmap.
