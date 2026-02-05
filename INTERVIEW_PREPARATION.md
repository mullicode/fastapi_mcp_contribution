# FastAPI-MCP: Technical Interview Preparation Guide

## Project Overview

**FastAPI-MCP** is a Python library that automatically converts FastAPI REST API endpoints into Model Context Protocol (MCP) tools, enabling seamless integration between FastAPI applications and AI/LLM systems. The project bridges the gap between traditional REST APIs and the emerging MCP standard, allowing developers to expose their APIs to AI agents with minimal configuration.

**Key Value Proposition**: Zero-to-minimal configuration required - developers can add MCP capabilities to existing FastAPI applications with just a few lines of code, while maintaining full compatibility with FastAPI's authentication, dependency injection, and routing systems.

---

## 1. Technical Ownership & Depth of Contribution

### Core Architecture Decisions

#### **ASGI Transport Integration**
- **Decision**: Use FastAPI's native ASGI interface instead of HTTP calls
- **Impact**: Eliminates network overhead, enables unified infrastructure deployment
- **Technical Depth**: Implemented custom `httpx.ASGITransport` integration that allows the MCP server to communicate directly with FastAPI routes without making actual HTTP requests
- **Code Location**: `fastapi_mcp/server.py` lines 115-119

```python
self._http_client = http_client or httpx.AsyncClient(
    transport=httpx.ASGITransport(app=self.fastapi, raise_app_exceptions=False),
    base_url=self._base_url,
    timeout=10.0,
)
```

#### **Dual Transport Support (HTTP & SSE)**
- **Decision**: Implement both Streamable HTTP (new standard) and SSE (backwards compatibility)
- **Impact**: Supports latest MCP spec while maintaining compatibility with existing clients
- **Technical Depth**: Created separate transport layers (`FastApiHttpSessionManager`, `FastApiSseTransport`) with unified interface
- **Code Locations**: 
  - `fastapi_mcp/transport/http.py` - HTTP transport implementation
  - `fastapi_mcp/transport/sse.py` - SSE transport implementation

#### **OpenAPI Schema Conversion Engine**
- **Decision**: Build custom OpenAPI-to-MCP tool converter instead of using generic converters
- **Impact**: Preserves FastAPI-specific features (descriptions, examples, response schemas)
- **Technical Depth**: Complex schema resolution, parameter extraction (path/query/header/body), response schema generation
- **Code Location**: `fastapi_mcp/openapi/convert.py` - 268 lines of conversion logic

**Key Features Implemented**:
- Schema reference resolution (`resolve_schema_references`)
- Example generation from schemas (`generate_example_from_schema`)
- Response schema documentation with examples
- Parameter type inference and validation

### Authentication & Security Architecture

#### **OAuth 2.1 / MCP 2025-03-26 Spec Compliance**
- **Decision**: Implement full OAuth proxy system for MCP authentication
- **Impact**: Enables secure, standards-compliant authentication for AI agents
- **Technical Depth**: 
  - OAuth metadata proxy with dynamic client registration
  - Authorization endpoint proxy with scope/audience handling
  - FastAPI-native dependency injection for auth checks
- **Code Location**: `fastapi_mcp/auth/proxy.py` - 267 lines

**Key Innovation**: "Fake" dynamic client registration endpoint that simplifies OAuth setup for most use cases while maintaining spec compliance.

### Advanced Features

#### **Operation Filtering System**
- **Implementation**: Tag-based and operation ID-based filtering
- **Code Location**: `fastapi_mcp/server.py` `_filter_tools()` method (lines 594-656)
- **Use Case**: Allows selective exposure of API endpoints as MCP tools

#### **Header Forwarding**
- **Implementation**: Configurable header allowlist for forwarding auth tokens
- **Code Location**: `fastapi_mcp/server.py` lines 533-538
- **Use Case**: Maintains authentication context from MCP requests to API calls

---

## 2. Creativity & Innovation

### Problem-Solving Innovations

#### **1. FastAPI-Native Approach (vs. Generic Converter)**
- **Challenge**: Most MCP tools are generic OpenAPI converters that lose FastAPI-specific features
- **Solution**: Direct integration with FastAPI's OpenAPI generation, preserving:
  - Dependency injection for authentication
  - Response model schemas
  - Endpoint documentation
  - Example responses

#### **2. ASGI Transport (Zero Network Overhead)**
- **Challenge**: MCP servers typically make HTTP calls to APIs, adding latency and complexity
- **Solution**: Use ASGI transport to call FastAPI routes directly in-process
- **Benefit**: 
  - No network overhead
  - Unified deployment (same process)
  - Better error handling
  - Preserves request context

#### **3. Flexible Mounting System**
- **Innovation**: MCP server can be mounted to:
  - Same FastAPI app (unified deployment)
  - Separate FastAPI app (microservices architecture)
  - APIRouter (modular routing)
- **Code Location**: `mount_http()` and `mount_sse()` methods support optional router parameter

#### **4. OAuth Proxy Pattern**
- **Innovation**: Instead of requiring full OAuth implementation, provides proxy endpoints that:
  - Fetch metadata from existing OAuth providers
  - Add MCP-required endpoints (dynamic registration)
  - Handle missing parameters (scopes, audience) with defaults
- **Real-World Impact**: Works with Auth0, Okta, and other providers without modification

### Design Patterns Used

1. **Adapter Pattern**: Converting OpenAPI operations to MCP tools
2. **Proxy Pattern**: OAuth metadata and authorization proxies
3. **Factory Pattern**: Tool creation from OpenAPI schemas
4. **Strategy Pattern**: Multiple transport implementations (HTTP/SSE)

---

## 3. Real-World Engineering Problems Solved

### Problem 1: Schema Complexity & Reference Resolution
**Challenge**: OpenAPI schemas use `$ref` references that need resolution before conversion
**Solution**: Implemented recursive reference resolver that handles:
- Circular references
- Nested references
- Multiple schema files
**Code**: `fastapi_mcp/openapi/utils.py` - `resolve_schema_references()`

### Problem 2: Parameter Type Inference
**Challenge**: OpenAPI parameters may not have explicit types
**Solution**: Implemented type inference from schema properties:
- Array types → list
- Object types → dict
- Primitive types → string/int/float/bool
**Code**: `fastapi_mcp/openapi/utils.py` - `get_single_param_type_from_schema()`

### Problem 3: Session Management in Async Context
**Challenge**: MCP HTTP transport requires session management in async FastAPI context
**Solution**: 
- Lazy initialization of session manager
- Background task management with proper cleanup
- Thread-safe startup with asyncio locks
**Code**: `fastapi_mcp/transport/http.py` - `_ensure_session_manager_started()`

### Problem 4: Request Context Preservation
**Challenge**: Need to forward HTTP request info (headers, cookies) from MCP requests to API calls
**Solution**: Extract request context from MCP server's request context and forward allowed headers
**Code**: `fastapi_mcp/server.py` lines 154-184

### Problem 5: Backwards Compatibility
**Challenge**: MCP spec evolved from SSE to HTTP transport
**Solution**: 
- Maintained both transports
- Deprecated old `mount()` method with clear migration path
- Provided examples for both approaches

### Problem 6: Error Handling & Debugging
**Challenge**: Complex conversion logic needs clear error messages
**Solution**: 
- Comprehensive logging throughout
- Validation errors with context
- Graceful degradation (skip invalid operations)
**Code**: Multiple logger statements throughout conversion logic

---

## 4. Fullstack Capabilities Demonstration

### Backend/API Design ✅

#### **RESTful API Design**
- **Evidence**: Works with any FastAPI application following REST principles
- **Examples**: `examples/shared/apps/items.py` demonstrates CRUD operations
- **Features**:
  - GET, POST, PUT, DELETE, PATCH support
  - Path parameters, query parameters, request bodies
  - Response models with validation

#### **API Architecture Patterns**
- **Microservices Support**: Can mount MCP server separately from API (`examples/04_separate_server_example.py`)
- **Modular Routing**: Supports APIRouter for organized endpoints
- **Dependency Injection**: Leverages FastAPI's `Depends()` for auth and business logic

#### **API Documentation**
- **OpenAPI/Swagger Integration**: Automatically generates OpenAPI schemas
- **Response Documentation**: Includes response schemas, examples, and status codes in tool descriptions

### Database Usage ⚠️ (Indirect)

**Note**: This library doesn't directly use databases, but it integrates with FastAPI applications that do.

#### **Database Integration Patterns**
- **Evidence**: Examples show in-memory databases (`items_db` dictionary)
- **Real-World Usage**: FastAPI apps using this library typically integrate with:
  - SQLAlchemy (SQL databases)
  - MongoDB (NoSQL)
  - Redis (caching)
  - Any database via FastAPI's dependency injection

#### **Data Persistence Considerations**
- **Session Management**: HTTP transport includes session storage (`EventStore`)
- **State Management**: MCP sessions can maintain state across tool calls
- **Code**: `fastapi_mcp/transport/http.py` - session manager with optional event store

**How to Present**: "While FastAPI-MCP doesn't directly interact with databases, it's designed to work seamlessly with FastAPI applications that use any database. The library preserves database transactions and sessions through FastAPI's dependency injection system, allowing authenticated MCP tool calls to access the same database connections and business logic as regular API endpoints."

### Infrastructure ✅

#### **CI/CD Pipeline**
- **Location**: `.github/workflows/ci.yml`
- **Features**:
  - Multi-version Python testing (3.10, 3.11, 3.12)
  - Linting with Ruff
  - Type checking with MyPy
  - Test coverage reporting (Codecov)
  - Automated dependency management with `uv`

#### **Package Distribution**
- **PyPI Package**: Published as `fastapi-mcp` (version 0.4.0)
- **Build System**: Hatchling (modern Python packaging)
- **Dependencies**: Managed with `uv.lock` for reproducible builds
- **Code**: `pyproject.toml` - complete package configuration

#### **Deployment Options**
- **Unified Deployment**: MCP server and API in same process
- **Separate Deployment**: MCP server as separate service
- **Documentation**: `docs/advanced/deploy.mdx` covers deployment strategies
- **Code**: `examples/04_separate_server_example.py`

#### **Development Infrastructure**
- **Pre-commit Hooks**: Automated code quality checks
- **Type Safety**: Full type hints with MyPy validation
- **Code Quality**: Ruff for linting and formatting
- **Documentation**: Comprehensive docs with examples

#### **Testing Infrastructure**
- **Test Coverage**: 9 test files covering:
  - Basic functionality
  - OpenAPI conversion
  - HTTP/SSE transports
  - Authentication
  - Complex applications
- **Test Framework**: pytest with async support
- **Fixtures**: Reusable test fixtures for FastAPI apps

### Frontend Development ❌

**Note**: This is a backend library with no frontend components.

**How to Present**: "FastAPI-MCP is a backend library focused on API-to-AI integration. While it doesn't include frontend code, it enables frontend developers to build AI-powered applications by exposing backend APIs as MCP tools. The library generates comprehensive API documentation that frontend developers can use, and it supports CORS and authentication patterns that frontend applications require."

**Alternative Angle**: If you have frontend experience, you could discuss:
- How MCP clients (like Claude Desktop) act as the "frontend" consuming these tools
- How the library enables building AI-powered UIs
- The JSON-RPC protocol used for client-server communication

---

## 5. Key Technical Highlights for Discussion

### Async/Await Patterns
- **Evidence**: Extensive use of `async/await` throughout
- **Examples**: 
  - `async def handle_call_tool()` - async tool execution
  - `async def handle_fastapi_request()` - async request handling
  - Background task management with `asyncio.create_task()`

### Protocol Implementation
- **MCP Protocol**: Full implementation of Model Context Protocol specification
- **JSON-RPC**: Underlying message protocol
- **OAuth 2.1**: Authentication protocol implementation
- **OpenAPI 3.0**: Schema parsing and conversion

### Error Handling & Resilience
- **Graceful Degradation**: Skips invalid operations instead of failing
- **Comprehensive Logging**: Debug, info, warning, error levels
- **Exception Handling**: Proper exception propagation with context

### Performance Considerations
- **ASGI Transport**: Zero network overhead for in-process calls
- **Lazy Initialization**: Session managers start on first request
- **Efficient Schema Resolution**: Single-pass reference resolution

---

## 6. Interview Talking Points

### Opening Statement (30 seconds)
"I built FastAPI-MCP, a library that automatically converts FastAPI REST APIs into Model Context Protocol tools. It enables AI agents to interact with existing APIs with zero configuration, while preserving authentication, documentation, and all FastAPI features. The project demonstrates my ability to work with complex protocols, async Python, and build developer tools that solve real integration challenges."

### Technical Depth (2-3 minutes)
"I designed the core architecture around three key decisions:
1. **ASGI Transport**: Instead of making HTTP calls, I use FastAPI's ASGI interface for zero-overhead in-process communication
2. **Native OpenAPI Conversion**: Built a custom converter that preserves FastAPI-specific features like dependency injection and response models
3. **Dual Transport Support**: Implemented both HTTP (new standard) and SSE (backwards compatibility) transports

The most complex part was the OpenAPI-to-MCP conversion engine, which handles schema reference resolution, parameter type inference, and generates comprehensive tool descriptions with examples."

### Real-World Problem Solving (2 minutes)
"One interesting challenge was session management in async contexts. The MCP HTTP transport requires a session manager that runs in a background task, but it needs to start lazily on the first request. I solved this with asyncio locks for thread-safe initialization and proper cleanup on shutdown.

Another challenge was OAuth integration. Instead of requiring users to implement full OAuth, I created proxy endpoints that fetch metadata from existing providers and add MCP-required endpoints, making it work with Auth0 and Okta out of the box."

### Fullstack Perspective (1-2 minutes)
"While this is primarily a backend library, it demonstrates fullstack thinking:
- **API Design**: Works with any REST API architecture, supporting microservices and modular routing
- **Infrastructure**: Complete CI/CD pipeline, PyPI distribution, and deployment documentation
- **Developer Experience**: Zero-configuration approach, comprehensive examples, and clear error messages

The library integrates with FastAPI applications that use databases, so while it doesn't directly interact with databases, it preserves database transactions through FastAPI's dependency injection."

### Creativity & Innovation (1 minute)
"The key innovation is the FastAPI-native approach. Most MCP tools are generic OpenAPI converters, but I built direct integration that preserves FastAPI's dependency injection for authentication. This means developers can use their existing `Depends()` functions without modification, making adoption seamless."

### Metrics & Impact
- **PyPI Package**: Published and maintained
- **Test Coverage**: Comprehensive test suite with 9 test files
- **Documentation**: Full documentation site with examples
- **CI/CD**: Automated testing across Python 3.10-3.12
- **Community**: Open source with contribution guidelines

---

## 7. Potential Questions & Answers

### Q: "Why did you choose to build this instead of using existing solutions?"
**A**: "Existing MCP tools were generic OpenAPI converters that lost FastAPI-specific features like dependency injection and response models. I wanted a solution that felt native to FastAPI developers, preserving their existing authentication and business logic without modification."

### Q: "What was the hardest technical challenge?"
**A**: "The OpenAPI schema conversion was complex because schemas use `$ref` references that can be circular or deeply nested. I had to build a robust reference resolver that handles edge cases while maintaining performance. Another challenge was implementing session management in async contexts with proper cleanup."

### Q: "How does this scale?"
**A**: "The ASGI transport eliminates network overhead, so it scales with the underlying FastAPI application. The library supports separate deployment for microservices architectures, and the session management is designed to handle concurrent requests efficiently."

### Q: "What would you improve?"
**A**: "I'd add:
1. WebSocket transport support for real-time updates
2. Caching layer for frequently accessed schemas
3. Metrics and observability hooks
4. GraphQL API support in addition to REST"

### Q: "How do you handle authentication?"
**A**: "The library supports OAuth 2.1 compliant with MCP 2025-03-26 spec. It can proxy existing OAuth providers or use custom metadata. Most importantly, it integrates with FastAPI's `Depends()` system, so developers use their existing authentication without changes."

---

## 8. Code Examples to Reference

### Basic Usage (Show Simplicity)
```python
from fastapi import FastAPI
from fastapi_mcp import FastApiMCP

app = FastAPI()
mcp = FastApiMCP(app)
mcp.mount_http()  # That's it!
```

### Authentication Integration (Show Real-World Usage)
```python
from fastapi import Depends, HTTPException
from fastapi_mcp import FastApiMCP, AuthConfig

async def authenticate(request: Request):
    # Your existing auth logic
    if not valid_token:
        raise HTTPException(401)

mcp = FastApiMCP(
    app,
    auth_config=AuthConfig(dependencies=[Depends(authenticate)])
)
```

### Advanced: Separate Deployment (Show Architecture Understanding)
```python
# api_app.py - Your main API
api_app = FastAPI()
# ... define endpoints ...

# mcp_app.py - Separate MCP server
mcp_app = FastAPI()
mcp = FastApiMCP(api_app)  # Create from API app
mcp.mount_http(mcp_app)     # Mount to separate app
```

---

## 9. Project Statistics

- **Lines of Code**: ~2,000+ lines across core modules
- **Test Coverage**: 9 test files with comprehensive coverage
- **Dependencies**: Well-managed with modern Python tooling (`uv`)
- **Documentation**: Full docs site + README + examples
- **Version**: 0.4.0 (actively maintained)
- **Python Versions**: Supports 3.10, 3.11, 3.12
- **License**: MIT (open source)

---

## 10. Closing Statement

"This project demonstrates my ability to:
1. **Understand complex protocols** (MCP, OAuth, OpenAPI) and implement them correctly
2. **Design developer-friendly APIs** with zero-configuration defaults
3. **Solve real integration challenges** (ASGI transport, OAuth proxying)
4. **Build production-ready software** (testing, CI/CD, documentation, packaging)
5. **Think about infrastructure** (deployment options, scalability, security)

It's a practical solution to a real problem: making existing APIs accessible to AI agents without rewriting code or losing existing features."

---

## Additional Preparation Tips

1. **Run the Examples**: Be ready to demo the library working
2. **Review Recent Changes**: Check CHANGELOG.md for latest features
3. **Know the MCP Spec**: Understand what MCP is and why it matters
4. **Prepare Architecture Diagram**: Be ready to draw the system architecture
5. **Discuss Trade-offs**: Be prepared to discuss design decisions and alternatives

---

## Resources to Reference

- **GitHub**: https://github.com/tadata-org/fastapi_mcp
- **PyPI**: https://pypi.org/project/fastapi-mcp/
- **Documentation**: https://fastapi-mcp.tadata.com/
- **MCP Spec**: https://modelcontextprotocol.io/
