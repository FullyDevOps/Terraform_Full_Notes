### **3.1 Introduction to HashiCorp Configuration Language (HCL)**
*   **What it is**: HCL is a **domain-specific language (DSL)** created by HashiCorp specifically for **infrastructure-as-code (IaC)**. It's the primary language for writing Terraform configurations.
*   **Why HCL?**:
    *   **Human-Readable & Editable**: Prioritizes readability and writability by humans over machine efficiency (unlike pure JSON).
    *   **Structured Data**: Represents configuration as structured data (objects, lists, maps).
    *   **Expressions & Logic**: Supports variables, functions, conditionals, and loops for dynamic configurations.
    *   **Safety**: Designed with IaC principles in mind (idempotency, declarative syntax).
    *   **JSON Compatibility**: Has a strict JSON syntax subset (`*.tf.json`). Terraform converts all `.tf` files to this JSON format internally.
*   **HCL vs. JSON**:
    *   **HCL (`.tf`)**: Preferred for authoring. More concise, supports comments, interpolation syntax (legacy), cleaner syntax.
    *   **JSON (`.tf.json`)**: Machine-friendly. Useful for generated configs or when integrating with tools that output JSON. *Avoid manual editing unless necessary.*
*   **HCL2**: The version used by Terraform (v0.12+). Major improvements over HCL1:
    *   First-class expressions everywhere (no more `${}` for most things).
    *   Strict type system.
    *   Consistent collection handling.
    *   Improved function syntax.
    *   *Terraform v0.12 was a massive breaking change; all modern Terraform uses HCL2.*

---

### **3.2 Syntax Basics: Blocks, Arguments, Expressions**
The core building blocks of HCL.

1.  **Blocks**:
    *   **Purpose**: Declare *configurable objects* (resources, providers, modules, variables, outputs, locals, data sources) or *nested configuration sections* (like `lifecycle` inside a resource).
    *   **Syntax**:
        ```hcl
        BLOCK_TYPE "LABEL" ["LABEL"...] {
          # Block body (arguments & nested blocks)
          argument = expression
          nested_block {
            # ...
          }
        }
        ```
    *   **Components**:
        *   `BLOCK_TYPE`: Required (e.g., `resource`, `provider`, `variable`, `output`, `module`, `data`, `terraform`).
        *   `LABEL(s)`: Optional strings defining the *name* or *type* of the object. Number and meaning depend on `BLOCK_TYPE`.
            *   `resource "aws_instance" "web_server"`: `aws_instance` = resource type (1st label), `web_server` = resource name (2nd label).
            *   `variable "env"`: `env` = variable name (1st label).
            *   `provider "aws"`: `aws` = provider name (1st label).
        *   `{ ... }`: Body containing arguments and nested blocks.
    *   **Nested Blocks**: Blocks defined *within* another block's body (e.g., `lifecycle`, `ingress` inside `aws_security_group`).

2.  **Arguments**:
    *   **Purpose**: Assign values to *configuration parameters* within a block.
    *   **Syntax**: `NAME = EXPRESSION`
        *   `NAME`: Identifier (e.g., `ami`, `instance_type`, `count`).
        *   `EXPRESSION`: Value (literal, variable, function call, etc.).
    *   **Key Points**:
        *   Order of arguments *within a block* generally **does not matter** (Terraform resolves dependencies).
        *   Must be unique *within their block* (except for specific multi-argument blocks like `provisioner`).
        *   Case-sensitive (`instance_type` != `Instance_Type`).

3.  **Expressions**:
    *   **Purpose**: Represent *values* – literals, variables, results of functions, or combinations thereof.
    *   **Where Used**: Almost everywhere a value is needed (argument values, `for` expressions, conditionals).
    *   **Types** (Detailed in 3.5):
        *   **Literals**: Fixed values (`"string"`, `42`, `true`, `[1, 2, 3]`, `{"key"="value"}`).
        *   **Variables**: References to input variables (`var.env`), locals (`local.region`), or resource attributes (`aws_instance.web.private_ip`).
        *   **Functions**: Calls to built-in functions (`lower("HELLO")`, `concat(list1, list2)`).
        *   **Operators**: Math (`+`, `-`), logic (`&&`, `||`), comparison (`==`, `!=`), collection access (`list[index]`, `map.key` or `map["key"]`).

---

### **3.3 Terraform File Structure (.tf files)**
*   **File Extension**: `.tf` (HCL syntax) or `.tf.json` (JSON syntax). **Always use `.tf` for manual authoring.**
*   **File Agnosticism**: Terraform **merges all `.tf` files** in the working directory *before* processing. **Order of files does not matter.**
*   **Recommended Structure** (Convention, not enforced):
    *   `main.tf`: Core configuration (resources, modules).
    *   `variables.tf`: Input variable declarations (`variable "name" { ... }`).
    *   `outputs.tf`: Output value declarations (`output "name" { ... }`).
    *   `providers.tf`: Provider configuration blocks (`provider "aws" { ... }`).
    *   `locals.tf`: Local value definitions (`locals { ... }`).
    *   `versions.tf`: Terraform & required provider versions (`terraform { required_version = ">= 1.0" ... }`).
    *   `networking.tf`, `compute.tf`, etc.: Feature-based grouping.
*   **Why Structure Matters**:
    *   **Readability**: Easier for humans to navigate.
    *   **Maintainability**: Changes are localized.
    *   **Team Collaboration**: Clear separation of concerns.
    *   **Terraform State Clarity**: Resource addresses in state reflect module/file structure.
*   **Critical Note**: Terraform **ignores subdirectories** unless they are modules (`source = "./modules/vpc"`). Place all root module config directly in the working directory.

---

### **3.4 Comments, Whitespace, and Formatting**
*   **Comments**:
    *   **Single-Line**: `# This is a comment`
    *   **Multi-Line (HCL only)**: `/* This is a
        multi-line comment */`
    *   **Where Allowed**: Anywhere *outside* of expressions or string literals. *Cannot* be inside `${ }` interpolation (legacy) or within a string.
    *   **Purpose**: Documentation, explaining *why* (not *what*), temporarily disabling config (use cautiously!).
*   **Whitespace**:
    *   **Ignored**: Terraform ignores insignificant whitespace (spaces, tabs, newlines).
    *   **Significant**:
        *   Inside string literals (`"hello world"` vs `"helloworld"`).
        *   Separating identifiers/tokens (you need space between `resource` and `"aws_instance"`).
        *   Within multi-line strings (heredocs).
*   **Formatting**:
    *   **`terraform fmt`**: **MANDATORY TOOL**. Automatically reformats `.tf` files to HashiCorp's canonical style.
        *   Standardizes indentation (2 spaces, **no tabs**).
        *   Manages spacing around operators/braces.
        *   Sorts block labels consistently.
        *   Ensures consistent style across teams/projects.
    *   **Why Enforce Formatting?**:
        *   Eliminates "style wars" in PRs.
        *   Makes diffs meaningful (only show actual config changes, not whitespace).
        *   Improves readability significantly.
    *   **Best Practice**: Run `terraform fmt` **before every commit**. Integrate it into your IDE/pre-commit hooks.

---

### **3.5 Expressions: Literals, Variables, Functions**
Expressions compute values.

1.  **Literals**:
    *   **String**: `"hello"`, `'world'`, `"Multi-line\nstring"`, `<<EOT\nHeredoc\nEOT` (heredoc).
        *   Interpolation: `"Value: ${var.x}"` (v0.12+ only within strings; see 3.6).
        *   Escape sequences: `\n`, `\t`, `\"`, `\'`, `\\`.
    *   **Number**: `42`, `-1`, `3.14`, `6.022e23`. Integers and floats.
    *   **Boolean**: `true`, `false`.
    *   **List (Array)**: `["a", "b", "c"]`, `[1, 2, 3]`. Ordered, indexable (`list[0]`).
    *   **Map (Object)**: `{"key1" = "value1", "key2" = "value2"}`, `{name = "Alice", age = 30}`. Key-value pairs, key-based access (`map.key` or `map["key"]`).
    *   **Set**: `{ "a", "b", "c" }` (v0.14+). Unordered, unique elements. *Cannot* have duplicates. Accessed via `for` expressions.
    *   **Null**: `null`. Represents absence of a value.

2.  **Variables**:
    *   **Input Variables (`var.*`)**: Values passed *into* the module (declared in `variables.tf`). `var.env`.
    *   **Local Values (`local.*`)**: Named intermediate values *within* a module (defined in `locals { ... }`). `local.region`.
    *   **Resource Attributes (`<RESOURCE_TYPE>.<NAME>.<ATTRIBUTE>`)**: Output values *from* resources. `aws_instance.web.private_ip`.
    *   **Data Source Attributes (`data.<TYPE>.<NAME>.<ATTRIBUTE>`)**: Output values *from* data sources. `data.aws_ami.ubuntu.id`.
    *   **Output Values (`module.<MODULE_NAME>.<OUTPUT_NAME>`)**: Values exposed *by* child modules. `module.vpc.private_subnets`.
    *   **Path Variables (`path.*`)**: `path.root`, `path.module`, `path.cwd` (current working directory).
    *   **Terraform Variables (`terraform.*`)**: Limited set (e.g., `terraform.workspace`).

3.  **Functions**:
    *   **Purpose**: Perform operations on values (transform, query, manipulate).
    *   **Syntax**: `function_name(arg1, arg2, ...)`
    *   **Common Categories**:
        *   **String**: `lower()`, `upper()`, `substr()`, `replace()`, `format()`, `join()`, `split()`.
        *   **Collection**: `length()`, `element()`, `coalesce()`, `setproduct()`, `zipmap()`, `keys()`, `values()`.
        *   **Encoding**: `jsonencode()`, `yamlencode()`, `base64encode()`, `csvencode()`.
        *   **File**: `file()`, `fileexists()`, `fileset()`, `templatefile()`.
        *   **Date/Time**: `timestamp()`, `formatdate()`.
        *   **Numeric**: `ceil()`, `floor()`, `log()`, `max()`, `min()`.
        *   **Type Conversion**: `tonumber()`, `tostring()`, `tobool()`, `toset()`, `tolist()`, `tomap()`.
        *   **IP Network**: `cidrhost()`, `cidrsubnet()`, `cidrsubnets()`.
    *   **Key Point**: Functions are **pure** – they only depend on inputs, no side effects.

---

### **3.6 Interpolation Syntax `${...}` and Bare Expressions (v0.12+)**
*   **The Great Change (v0.12)**: Terraform v0.12 **eliminated the requirement** to wrap *all* expressions in `${ }`.
*   **Bare Expressions (v0.12+)**:
    *   **Where Used**: **Directly** as argument values, within `for` expressions, conditionals, etc.
    *   **Syntax**: Just write the expression.
    *   **Example**:
        ```hcl
        # v0.11 (HCL1) - REQUIRED ${}
        instance_type = "${var.instance_type}"

        # v0.12+ (HCL2) - BARE EXPRESSION (CORRECT)
        instance_type = var.instance_type

        # v0.12+ - BARE EXPRESSION in complex case
        tags = {
          Environment = var.env
          Owner       = lower(var.owner)
        }
        ```
*   **`${...}` Interpolation (Legacy/Strings Only)**:
    *   **Where Used**: **ONLY** within **double-quoted strings** (`"..."`) to **embed expressions**.
    *   **Purpose**: String interpolation (like f-strings in Python).
    *   **Syntax**: `"Literal text ${expression} more literal text"`
    *   **Example**:
        ```hcl
        # Interpolation ONLY inside double quotes
        name = "web-server-${count.index}"

        # WRONG (v0.12+): instance_type = ${var.type}  (No quotes! Use bare expression)
        # CORRECT (v0.12+): instance_type = var.type

        # CORRECT (v0.12+): Using interpolation inside a string
        user_data = <<-EOT
          #!/bin/bash
          echo "Hello from ${aws_instance.web.id}"
        EOT
        ```
*   **Critical Rules**:
    1.  **Never** use `${ }` for argument values outside of strings (v0.12+). It causes syntax errors.
    2.  **Always** use bare expressions for non-string argument values.
    3.  **Only** use `${ }` when you need to *dynamically insert* a value *into a string literal*.
    4.  **Single-quoted strings** (`'...'`) **do not** support interpolation. Use double quotes for interpolation.

---

### **3.7 Conditional Expressions (Ternary)**
The **only** inline conditional operator in HCL. Syntax: `CONDITION ? TRUE_VAL : FALSE_VAL`

*   **Purpose**: Choose between two values based on a boolean condition.
*   **Syntax**: `condition ? true_expression : false_expression`
*   **Evaluation**:
    1.  `condition` is evaluated (must result in `true` or `false`).
    2.  If `condition` is `true`, `true_expression` is evaluated and returned.
    3.  If `condition` is `false`, `false_expression` is evaluated and returned.
    4.  *Only* the selected branch is evaluated (short-circuiting).
*   **Common Use Cases**:
    *   Environment-specific configuration.
    *   Enabling/disabling features conditionally.
    *   Setting values based on boolean variables.
*   **Examples**:
    ```hcl
    # Simple boolean variable
    count = var.create_instance ? 1 : 0

    # Based on workspace (common pattern)
    instance_type = terraform.workspace == "prod" ? "m5.large" : "t3.small"

    # Nested ternary (use sparingly for readability)
    size = var.env == "prod" ? "large" : (var.env == "staging" ? "medium" : "small")

    # Using in resource attribute
    resource "aws_instance" "web" {
      ami           = var.os == "linux" ? "ami-0c7217cdde317cfec" : "ami-0b0af3577fe5e3532"
      instance_type = var.env == "prod" ? "m5.large" : "t3.micro"
    }
    ```
*   **Critical Nuances**:
    *   **Type Consistency**: `true_expression` and `false_expression` **must be convertable to the same type**. Terraform will try to convert, but mismatched types cause errors (e.g., `string` vs `number`).
    *   **Avoid Complex Logic**: Keep conditions simple. For complex branching, use `locals` with `for` expressions or separate resources/modules.
    *   **Not for Control Flow**: Cannot conditionally *create* entire blocks (use `count`/`for_each` on the block itself instead).

---

### **3.8 Collections: Lists, Maps, Sets**
Structured data types for grouping values.

1.  **Lists (Arrays)**:
    *   **Purpose**: Ordered sequence of values (like arrays in most languages).
    *   **Syntax**: `[value1, value2, value3]`
    *   **Access**: By **zero-based index** (`list[0]`).
    *   **Key Properties**:
        *   **Ordered**: `[1, 2]` != `[2, 1]`.
        *   **Duplicates Allowed**: `[1, 1, 2]` is valid.
        *   **Mutable Order**: Changing element order creates a *different list*.
    *   **Example**:
        ```hcl
        variable "azs" {
          type    = list(string)
          default = ["us-east-1a", "us-east-1b", "us-east-1c"]
        }
        availability_zone = var.azs[0] # "us-east-1a"
        ```

2.  **Maps (Objects/Dictionaries)**:
    *   **Purpose**: Collection of **key-value pairs** (like dicts in Python, objects in JS).
    *   **Syntax**:
        *   `{ key1 = value1, key2 = value2 }` (Preferred for config)
        *   `{ "key1" = value1, "key2" = value2 }`
    *   **Access**:
        *   Dot notation (for valid identifiers): `map.key`
        *   Bracket notation (always works, esp. for dynamic keys): `map["key"]`
    *   **Key Properties**:
        *   **Keys are Strings**: Implicitly converted to strings if possible (`{1 = "a"}` becomes `{"1" = "a"}`).
        *   **Unique Keys**: Duplicate keys are invalid.
        *   **Unordered**: `{a=1, b=2}` == `{b=2, a=1}`.
    *   **Example**:
        ```hcl
        variable "instance_types" {
          type = map(string)
          default = {
            prod  = "m5.large"
            stage = "t3.medium"
            dev   = "t3.small"
          }
        }
        instance_type = var.instance_types[var.env] # e.g., "m5.large" if env="prod"
        ```

3.  **Sets**:
    *   **Purpose**: Unordered collection of **unique** elements (like sets in math).
    *   **Syntax**: `{ value1, value2, value3 }` (Note: **NO** `=` signs)
    *   **Access**: Primarily via `for` expressions (`[for s in set : ...]`). No direct index access.
    *   **Key Properties**:
        *   **Unordered**: `{1, 2}` == `{2, 1}`.
        *   **Unique Elements**: Duplicates are automatically removed (`{1, 1, 2}` becomes `{1, 2}`).
        *   **Element Type Must Be Consistent**: All elements must be the same type.
        *   **Cannot Contain Complex Types**: Elements must be primitive types (string, number, bool) or *homogeneous* collections (sets of strings, sets of maps with same schema). *Cannot* mix types.
    *   **Why Use Sets?**:
        *   Guarantee uniqueness (e.g., security group rules).
        *   Efficient membership checks (`value in set`).
        *   Set operations (`setunion()`, `setintersection()`, `setsubtract()`).
    *   **Example**:
        ```hcl
        # Input a list, convert to set to deduplicate & enable set ops
        variable "allowed_ips" {
          type    = set(string)
          default = ["10.0.0.0/24", "192.168.1.0/24", "10.0.0.0/24"] # Duplicate!
        }
        # allowed_ips becomes {"10.0.0.0/24", "192.168.1.0/24"} (order not guaranteed)

        resource "aws_security_group" "web" {
          ingress {
            from_port   = 80
            to_port     = 80
            protocol    = "tcp"
            cidr_blocks = var.allowed_ips # Unique IPs enforced
          }
        }
        ```

*   **Type Constraints**: Always define `type` in `variable` blocks (e.g., `type = list(string)`, `type = map(number)`, `type = set(object({id=string, name=string}))`). Prevents runtime errors.

---

### **3.9 Loops and Iteration (`for_each`, `count`)**
Control resource/module instantiation based on collections.

1.  **`count` Argument**:
    *   **Purpose**: Create **multiple, nearly identical** resource instances based on a **number**.
    *   **Syntax**: `count = <number>`
    *   **How it Works**:
        *   Creates `count` instances indexed from `0` to `count - 1`.
        *   Access index via `count.index` (e.g., `count.index` in `name = "server-${count.index}"`).
    *   **Use Case**: Simple scaling where instances are truly identical (e.g., 3 identical web servers).
    *   **Example**:
        ```hcl
        resource "aws_instance" "web" {
          count         = 3
          ami           = "ami-0c7217cdde317cfec"
          instance_type = "t3.micro"
          tags = {
            Name = "web-${count.index}" # "web-0", "web-1", "web-2"
          }
        }
        ```
    *   **Critical Limitations & Pitfalls**:
        *   **Index-Based**: If you remove an instance in the *middle* (e.g., set `count=2`), Terraform **destroys the *last* instance (`index=2`), NOT the one you might expect**. This causes **state drift and potential data loss**.
        *   **No Stable Identity**: Instances are identified by index, which changes if `count` changes. Bad for state stability.
        *   **Cannot Easily Modify Subset**: Hard to change attributes for just one instance.
        *   **Avoid for Dynamic Collections**: If your list/map of inputs changes size, `count` causes chaos.

2.  **`for_each` Argument**:
    *   **Purpose**: Create **multiple instances** where each instance is **associated with a unique key** from a **map or set** of strings.
    *   **Syntax**: `for_each = <map or set of strings>`
    *   **How it Works**:
        *   Iterates over each key-value pair in a **map** (`for_each = { key1 = val1, key2 = val2 }`) OR each element in a **set** (`for_each = toset(["a", "b", "c"])`).
        *   Access key via `each.key`, value via `each.value` (for maps) or `each.key` (for sets, equals the element).
    *   **Use Case**: Creating resources based on a dynamic set of inputs where each resource has distinct configuration (e.g., security groups per environment, instances per AZ).
    *   **Example (Map)**:
        ```hcl
        variable "instances" {
          type = map(object({
            instance_type = string
            count         = number
          }))
          default = {
            web  = { instance_type = "t3.micro", count = 2 }
            db   = { instance_type = "m5.large", count = 1 }
          }
        }

        resource "aws_instance" "app" {
          for_each      = var.instances
          ami           = "ami-0c7217cdde317cfec"
          instance_type = each.value.instance_type
          tags = {
            Role = each.key # "web" or "db"
          }
          # To get multiple instances *per* map entry, you'd need nested count/for_each (often messy)
        }
        ```
    *   **Example (Set - Common for Tags/Names)**:
        ```hcl
        variable "azs" {
          type    = set(string)
          default = ["us-east-1a", "us-east-1b", "us-east-1c"]
        }

        resource "aws_instance" "web" {
          for_each      = var.azs
          ami           = "ami-0c7217cdde317cfec"
          instance_type = "t3.micro"
          availability_zone = each.key # "us-east-1a", etc.
          tags = {
            Name = "web-${each.key}"
          }
        }
        ```
    *   **Critical Advantages over `count`**:
        *   **Stable Identity**: Each instance has a **permanent, unique key** (`each.key`). Adding/removing items from the map/set only affects *that specific instance*. No state drift for unrelated instances.
        *   **Explicit Configuration**: Each instance's config is clearly defined by its key/value.
        *   **Handles Dynamic Collections Safely**: Perfect for when your input list/map changes.
    *   **Critical Requirements**:
        *   **Keys Must Be Stable & Unique**: Keys must not change between applies (e.g., don't use `uuid()`). Use meaningful names (AZ names, environment names).
        *   **Map/Set of Strings**: `for_each` requires a map with string keys OR a set of strings. Use `toset()`, `tomap()`, `keys()`, `values()` to convert.
        *   **Avoid `count` with `for_each`**: Don't nest them unnecessarily. If you need multiple identical instances *per* key, reconsider your design (e.g., use `count` *inside* the resource definition if absolutely necessary, but often indicates a design flaw).

*   **Which to Use?**
    *   **Use `for_each` (Almost Always)**: When instances have *any* distinguishing configuration (type, name, tags, etc.). It's safer and more maintainable.
    *   **Use `count` (Rarely)**: Only when *every single attribute* of *every instance* is *truly identical* except for an index-based name (very uncommon in real-world configs). Prefer `for_each` even for simple scaling if you can generate stable keys (e.g., `for_each = toset(range(0, 3))`).

---

### **3.10 Dynamic Blocks**
Generate **repetitive nested configuration blocks** dynamically within a resource/module block.

*   **Purpose**: Avoid manual repetition of nested blocks (like `ingress`, `egress`, `lifecycle`, `setting`) when the number and content depend on a variable collection.
*   **Problem Solved**: Before `dynamic`, you had to use complex `count` tricks inside resources or duplicate blocks, which was error-prone and hard to maintain.
*   **Syntax**:
    ```hcl
    resource "aws_security_group" "web" {
      # ... other arguments ...

      dynamic "ingress" {
        for_each = var.ingress_rules
        content {
          from_port   = ingress.value.from_port
          to_port     = ingress.value.to_port
          protocol    = ingress.value.protocol
          cidr_blocks = ingress.value.cidr_blocks
        }
      }
    }
    ```
*   **Components**:
    *   `dynamic "NESTED_BLOCK_TYPE"`: Declares the type of nested block to generate (e.g., `"ingress"`, `"setting"`, `"lifecycle"`).
    *   `for_each = <collection>`: A map or set defining the iterations (same rules as top-level `for_each`).
    *   `content { ... }`: The body of *each* generated nested block. Uses `dynamic_iterator` (`ingress` in the example) to access the current element.
        *   `iterator.value`: The value for the current iteration (if `for_each` is a map).
        *   `iterator.key`: The key for the current iteration (if `for_each` is a map) or the element (if `for_each` is a set).
*   **How it Works**:
    1.  Terraform evaluates the `for_each` expression (a map/set).
    2.  For **each element** in that collection:
        *   Creates one nested block of the specified type (`ingress`).
        *   Evaluates the `content` block, where `iterator` (e.g., `ingress`) provides access to the current key/value.
    3.  The generated nested blocks are inserted into the parent resource block.
*   **Example (Security Group Rules)**:
    ```hcl
    variable "ingress_rules" {
      type = list(object({
        from_port   = number
        to_port     = number
        protocol    = string
        cidr_blocks = list(string)
      }))
      default = [
        { from_port = 80, to_port = 80, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
        { from_port = 443, to_port = 443, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
      ]
    }

    resource "aws_security_group" "web" {
      name        = "web-sg"

      # DYNAMIC BLOCK for ingress rules
      dynamic "ingress" {
        for_each = var.ingress_rules
        content {
          from_port   = ingress.value.from_port
          to_port     = ingress.value.to_port
          protocol    = ingress.value.protocol
          cidr_blocks = ingress.value.cidr_blocks
        }
      }

      egress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
      }
    }
    ```
*   **Critical Nuances & Best Practices**:
    *   **Replaces Repetition**: Use instead of manually writing multiple `ingress { ... }` blocks.
    *   **Not for Top-Level Resources**: Only generates *nested* blocks *within* a single resource/module block. Use `for_each` on the resource itself to create multiple resources.
    *   **Iterator Name**: The name after `dynamic` (e.g., `ingress`) is arbitrary but must match what's used in `content`. Convention is to use the nested block type name.
    *   **Type Safety**: Define the variable type strictly (e.g., `list(object({...}))`) to catch errors early.
    *   **Complexity**: Keep the `content` block simple. If logic gets complex, consider using a `locals` map to pre-process the input.
    *   **When NOT to Use**: If you only ever have 1 or 2 fixed nested blocks, static blocks are clearer. Use `dynamic` when the number/rules are truly dynamic/varying.
    *   **Provider Support**: Not all providers/resources support all nested block types dynamically. Check provider documentation.

---

**Key Takeaways for Future Reference**:

1.  **HCL is Purpose-Built**: For safe, readable IaC. Prefer `.tf` files.
2.  **Structure Matters**: Use `main.tf`, `variables.tf`, `outputs.tf` convention.
3.  **`terraform fmt` is Non-Negotiable**: Enforce consistent style.
4.  **Bare Expressions Rule (v0.12+)**: `${}` **only** inside strings.
5.  **`for_each` > `count`**: Use `for_each` with stable keys for almost all multi-resource scenarios. Avoid `count`'s state drift.
6.  **Dynamic Blocks = Nested Repetition**: Generate `ingress`/`egress`/etc. blocks from collections.
7.  **Type Strictness**: Define `type` for variables/locals/outputs religiously.
8.  **Ternary is for Values**: Not for control flow; keep conditions simple.
9.  **Collections Have Semantics**: Lists (ordered, dupes), Maps (key-value), Sets (unique, unordered).
10. **Interpolation is String-Only**: `${expr}` only works within `"double quotes"`.
