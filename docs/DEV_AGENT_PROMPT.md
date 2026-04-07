# templ4docx - Development Agent Prompt

You are an AI development agent working on **templ4docx**, a lightweight Java utility library for programmatic manipulation of Microsoft Word DOCX files. Your task is to maintain code quality, implement features, and fix bugs while adhering to the existing architecture and coding patterns.

## Project Overview

**templ4docx** is a utility library built on top of Apache POI that enables developers to:
- Read DOCX file content as plain text
- Extract and identify variables within templates (via configurable regex patterns)
- Fill DOCX templates with dynamic data (text, images, tables, lists)
- Save the populated document as a new DOCX file

**Key Use Cases:**
- Automated document generation (contracts, reports, certificates)
- Mail merge functionality
- Template-based document creation
- Bulk document processing

### Core Technologies
- **Java** 1.6+ (Java 6+, backward compatible)
- **Apache POI 5.4.0** (XWPF for Word document manipulation)
- **Maven** (build and dependency management)
- **Apache Commons Lang3 3.18.0** (utilities)
- **Apache Commons IO 2.14.0** (file I/O)
- **JUnit 4.13.1** (testing)
- **FEST-Assert 1.4** (fluent assertion API)

## File Structure

```
templ4docx/
├── src/main/java/pl/jsolve/templ4docx/
│   ├── core/
│   │   ├── Docx.java              # Main entry point for template operations
│   │   └── VariablePattern.java    # Configurable delimiter/pattern (e.g., #{...}, ${...})
│   ├── executor/
│   │   └── DocumentExecutor.java   # Orchestrates variable substitution
│   ├── extractor/
│   │   └── VariablesExtractor.java # Finds variables in document using regex
│   ├── insert/
│   │   ├── Insert.java             # Abstract base for variable replacements
│   │   ├── TextInsert.java         # Text variable insertion
│   │   ├── ImageInsert.java        # Image variable insertion
│   │   ├── BulletListInsert.java   # Bullet list insertion
│   │   └── TableRowInsert.java     # Table row insertion
│   ├── variable/
│   │   ├── Variables.java          # Container for all variables to insert
│   │   ├── TextVariable.java       # Text variable definition
│   │   ├── ImageVariable.java      # Image variable definition
│   │   ├── BulletListVariable.java # Bullet list variable definition
│   │   └── TableVariable.java      # Table variable definition
│   ├── cleaner/
│   │   └── DocumentCleaner.java    # Merges split variables across runs/paragraphs
│   ├── meta/
│   │   └── DocumentMetaProcessor.java # Marks variable occurrences for future updates
│   ├── util/
│   │   └── Key.java                # Represents a variable occurrence location
│   ├── exception/
│   │   └── OpenDocxException.java  # Custom exception wrapper
│   └── sweetener/
│       └── ... (resource utilities)
├── src/test/java/...               # Unit and integration tests
├── pom.xml                          # Maven configuration
├── build.gradle                     # Gradle support (alternative build)
├── README.md                        # User documentation
├── docs/
│   └── DEV_AGENT_PROMPT.md         # This file
└── .gitignore
```

## Architecture & Design Patterns

### 1. Core Workflow

The typical usage flow:

```java
// 1. Open template
Docx docx = new Docx("template.docx");

// 2. Set variable delimiters (optional, default is ${...})
docx.setVariablePattern(new VariablePattern("#{", "}"));

// 3. Find all variables in template
List<String> variables = docx.findVariables();

// 4. Create Variables container and populate
Variables variables = new Variables();
variables.addTextVariable(new TextVariable("#{firstname}", "John"));
variables.addImageVariable(new ImageVariable("#{logo}", "logo.png"));

// 5. Fill template
docx.fillTemplate(variables);

// 6. Save result
docx.save("output.docx");
```

### 2. Key Classes

**Docx.java** (Entry Point)
- Opens DOCX files (from file path or InputStream)
- Manages XWPFDocument lifecycle
- Provides `findVariables()` for pattern extraction
- Coordinates filling via `fillTemplate(Variables)`
- Handles save operations (file path or OutputStream)

**VariablePattern.java**
- Defines variable delimiters (prefix and suffix)
- Default: `${` and `}`
- Customizable to any pattern (e.g., `#{` and `}`, `[[` and `]]`)

**Variables.java** (Container)
- Accumulates all variables (text, images, lists, tables)
- Accessed via maps keyed by variable name

**Insert Classes** (Abstract Hierarchy)
- `Insert` (abstract base) - represents a single variable occurrence
- `TextInsert` - simple text replacement
- `ImageInsert` - image embedding
- `BulletListInsert` - list insertion
- `TableRowInsert` - table row insertion

**DocumentExecutor.java** (Orchestration)
- Iterates through document structure
- Finds variable matches using `Key` positions
- Delegates to appropriate `Insert` subclass
- Handles paragraph and table cell content

**VariablesExtractor.java** (Pattern Matching)
- Uses regex based on VariablePattern
- Extracts all variable names from document text
- Returns List<String> of found variable identifiers

**DocumentCleaner.java** (Pre-processing)
- POI splits variables across XML runs when document is modified
- Merges split variables back together before execution
- Essential for complex templates with formatting

### 3. Variable Pattern System

Variables are identified by configurable delimiters:

```java
// Built-in patterns
new VariablePattern("${", "}")      // Default
new VariablePattern("#{", "}")      // Alternative
new VariablePattern("[[", "]]")     // Another variant

// Pattern extraction
Pattern regex = new Pattern(prefix, suffix);
List<String> vars = extractor.extract(content, regex);
// Returns: ["firstname", "lastname", "logo", "table_data"]
```

### 4. Variable Types

**Text Variables**
```java
variables.addTextVariable(new TextVariable("#{name}", "John Doe"));
variables.addTextVariable(new TextVariable("#{date}", "2026-04-07"));
```

**Image Variables**
```java
variables.addImageVariable(new ImageVariable("#{logo}", "logo.png"));
variables.addImageVariable(new ImageVariable("#{signature}", new FileInputStream(...)));
```

**Bullet List Variables**
```java
BulletListVariable list = new BulletListVariable("#{items}");
list.addItem("Item 1");
list.addItem("Item 2");
variables.addBulletListVariable(list);
```

**Table Variables**
```java
TableVariable table = new TableVariable("#{employees}");
table.addRow(new Object[]{"John", "Developer", "$100k"});
table.addRow(new Object[]{"Jane", "Designer", "$90k"});
variables.addTableVariable(table);
```

## Coding Patterns & Conventions

### 1. Exception Handling

Use `OpenDocxException` for all document-related errors:

```java
try {
  docx = new XWPFDocument(inputStream);
} catch (Exception ex) {
  throw new OpenDocxException(ex.getMessage(), ex.getCause());
}
```

**Pattern**: Wrap checked exceptions as unchecked `OpenDocxException` for cleaner API.

### 2. Resource Management

Use try-finally for stream closure:

```java
XWPFWordExtractor extractor = null;
try {
  extractor = new XWPFWordExtractor(docx);
  return extractor.getText();
} finally {
  if (extractor != null) {
    Resources.closeStream(extractor);
  }
}
```

### 3. Insert Implementations

Each `Insert` subclass follows this pattern:

```java
public class CustomInsert extends Insert {
  
  public CustomInsert(Key key) {
    super(key);
  }

  public void insert(XWPFRun run, String value) {
    // Implementation specific to variable type
  }
}
```

### 4. Key Class (Variable Location Tracking)

`Key` represents where a variable occurs in the document:

```java
Key key = new Key(runIndex, paragraphIndex, value);
// Used by Insert to locate and replace content
```

### 5. Document Structure Navigation

Access XWPFDocument structure via:

```java
XWPFDocument docx = docx.getXWPFDocument();

// Paragraphs
for (XWPFParagraph paragraph : docx.getParagraphs()) {
  for (XWPFRun run : paragraph.getRuns()) {
    // Run is the atomic unit with text
  }
}

// Tables
for (XWPFTable table : docx.getTables()) {
  for (XWPFTableRow row : table.getRows()) {
    for (XWPFTableCell cell : row.getTableCells()) {
      // Cell contains paragraphs
    }
  }
}
```

## Development Workflow

### Building

**Maven:**
```bash
mvn clean install      # Build and install locally
mvn test              # Run unit tests
mvn package           # Package JAR
```

**Gradle:**
```bash
gradle build
gradle test
```

### Testing

- **Location**: `src/test/java/`
- **Framework**: JUnit 4
- **Assertions**: FEST-Assert (fluent API)
- **Pattern**: One test class per main class

Example test:

```java
public class DocxTest {
  
  private Docx docx;

  @Before
  public void setUp() {
    docx = new Docx("src/test/resources/template.docx");
  }

  @Test
  public void testFindVariables() {
    List<String> variables = docx.findVariables();
    assertThat(variables).contains("firstname", "lastname");
  }

  @Test
  public void testFillTemplate() {
    Variables vars = new Variables();
    vars.addTextVariable(new TextVariable("#{firstname}", "John"));
    docx.fillTemplate(vars);
    docx.save("output.docx");
    // Verify output file exists
  }
}
```

### Code Style

- **Naming**: camelCase for variables/methods, PascalCase for classes
- **Access Modifiers**: Private fields, public getters/setters
- **Javadoc**: Public classes and methods should have Javadoc
- **Formatting**: Standard Java conventions (2-space indent in Maven)
- **Serialization**: Important classes implement `Serializable` (e.g., Docx)

## Known Limitations & Considerations

1. **Java 1.6 Compatibility**: Code must support Java 1.6+. Avoid:
   - Diamond operator `<>`
   - Try-with-resources
   - Lambda expressions
   - Functional interfaces

2. **POI Version**: Tightly coupled to Apache POI 5.4.0
   - Check POI release notes before upgrades
   - XWPF API may change between versions

3. **Variable Merging**: DocumentCleaner handles split runs
   - Variables can be fragmented across XML runs
   - Cleaning must happen before execution
   - Performance impact on large documents

4. **Meta Information**: Optional feature for update tracking
   - Marked variables can be updated post-fill
   - Enabled via `setProcessMetaInformation(true)`
   - Adds memory overhead

5. **Table Limitations**: Table variables work but are complex
   - Template must have a marker row
   - Complex nested tables may have issues
   - Image insertion in table cells is limited

## Common Issues & Troubleshooting

**Issue**: Variables not found
- **Cause**: VariablePattern delimiters don't match document content
- **Solution**: Check pattern using `docx.setVariablePattern(...)`

**Issue**: Text corruption after fill
- **Cause**: DocumentCleaner not properly merging split runs
- **Solution**: Ensure `fillTemplate()` is called (it calls cleaner internally)

**Issue**: Images not embedding
- **Cause**: Path incorrect or file permissions
- **Solution**: Use absolute paths or InputStream, check file exists

**Issue**: Performance degradation
- **Cause**: Large document with many variables or meta processing
- **Solution**: Disable meta processing if not needed, process documents in batches

## REQUIREMENTS.md Status

**NOT FOUND**: This repository does not have a `docs/REQUIREMENTS.md` file. Consider creating one to document:
- Supported DOCX versions and Office compatibility
- Performance benchmarks for document sizes
- Feature roadmap and limitations
- Breaking changes between versions

## Build & Deployment

**Maven Central**: Published via Nexus OSS
- GroupId: `pl.jsolve`
- ArtifactId: `templ4docx`
- Current version: `2.0.3-SNAPSHOT`

**Release Process** (via `release` Maven profile):
```bash
mvn -P release clean deploy  # Requires GPG signing & Nexus credentials
```

## Notes for Agents

- This is a **mature, stable library** with focused scope (DOCX templating only)
- **Backward compatibility is critical**: Java 1.6+ support must be maintained
- **POI is the only heavy dependency**: Be cautious with upgrades
- **Document structure is complex**: Understand XWPFDocument hierarchy before modifying
- **Variable extraction is regex-based**: Pattern mismatches are common debugging issues
- **Test thoroughly with real DOCX files**: Unit tests alone won't catch all POI edge cases
- Review **existing Insert implementations** before adding new variable types
- **Meta information processing** is optional; understand its cost before enabling
