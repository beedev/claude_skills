---
allowed-tools: [Read, Grep, Glob, Write, Edit, Bash, Task, WebFetch]
description: "Write an engaging, personalized project briefing that explains the entire project in plain language"
---

# /sc:project-briefing - Personal Project Briefing

## Purpose
Generate a comprehensive, engaging project briefing document (`FOR[name].md`) that explains the entire project in plain, memorable language. This is not boring technical documentation—it's a story about the project that teaches and inspires.

## Usage
```
/sc:project-briefing [name] [--focus area] [--depth quick|standard|deep]
```

## Arguments
- `name` - Your name for the personalized filename (e.g., "Bharath" → `FOR_BHARATH.md`)
- `--focus` - Optional focus area (architecture, lessons, tech-stack, debugging)
- `--depth` - Level of detail: quick (overview), standard (balanced), deep (comprehensive)

## Output Structure

### 1. The Story (Opening Hook)
Start with WHY this project exists. What problem does it solve? Who benefits? Make the reader care before diving into technical details.

### 2. The Big Picture (Architecture)
Explain the technical architecture like you're drawing on a whiteboard:
- What are the major components and how do they talk to each other?
- Use analogies (e.g., "Think of the pipeline like a factory assembly line...")
- Include a mental model diagram in ASCII or describe it visually
- Explain the data flow from user action to response

### 3. The Codebase Map (Structure)
Guide the reader through the codebase like a tour guide:
- "If you want to change X, look in Y"
- Key directories and what lives in each
- The most important files and why they matter
- How the pieces connect (imports, dependencies, data flow)

### 4. The Tech Stack (Technologies)
For each major technology:
- What it is and what it does for us
- Why we chose it over alternatives
- Any quirks or gotchas to be aware of
- Links to the best learning resources

### 5. The Decision Log (Why We Built It This Way)
Document the key architectural decisions:
- What options we considered
- What we chose and WHY
- What trade-offs we accepted
- What we'd do differently with hindsight

### 6. The War Stories (Lessons Learned)
This is the gold—make it memorable:

**Bugs That Bit Us**
- The bug: What went wrong and how it manifested
- The hunt: How we tracked it down
- The fix: What we changed and why it worked
- The lesson: How to avoid this in the future

**Pitfalls to Avoid**
- Common mistakes in this codebase
- Things that look right but are wrong
- Performance traps
- Security considerations

**New Technologies Mastered**
- What was new to us
- The learning curve and how we climbed it
- Key "aha!" moments
- Resources that helped most

### 7. The Craft (Engineering Wisdom)
Capture how good engineers think:
- Patterns used and when to apply them
- Best practices demonstrated in this codebase
- Code review insights
- Testing strategies that work
- Debugging techniques that saved us
- Performance optimization approaches

### 8. The Quick Reference
End with practical quick-reference:
- Common commands cheat sheet
- Environment setup gotchas
- Deployment checklist
- "If X breaks, check Y" troubleshooting guide

## Writing Style Guidelines

### DO:
- Write like you're explaining to a smart friend over coffee
- Use analogies: "The retriever is like a librarian who knows exactly which shelf..."
- Include anecdotes: "We once spent 3 hours debugging only to find..."
- Be honest about mistakes: "In hindsight, we should have..."
- Use humor where appropriate
- Make technical concepts sticky and memorable
- Use headers, bullets, and formatting for scannability

### DON'T:
- Write like a textbook or API documentation
- Use jargon without explaining it
- Skip the "why" and only cover the "what"
- Make it dry and impersonal
- Assume the reader knows the context
- Write walls of unformatted text

## Execution Steps

1. **Explore the Codebase**
   - Read CLAUDE.md, README, and key documentation
   - Understand the directory structure
   - Identify the main entry points and data flows
   - Find configuration files and understand settings

2. **Trace the Architecture**
   - Map out how components connect
   - Identify the core abstractions and patterns
   - Understand the database schema and data models
   - Document external integrations and APIs

3. **Gather the Stories**
   - Look at git history for major changes and bug fixes
   - Check for TODO comments and technical debt markers
   - Review any existing documentation for lessons learned
   - Identify clever solutions and interesting patterns

4. **Write the Briefing**
   - Start with the hook—make them want to read more
   - Build understanding progressively
   - Include code snippets where they illuminate
   - Add diagrams (ASCII) where visual helps
   - End with actionable quick reference

5. **Review for Engagement**
   - Read it aloud—does it flow?
   - Is every section interesting?
   - Would someone remember this tomorrow?
   - Are the lessons actually useful?

## Example Opening

```markdown
# FOR_BHARATH.md - DBNotebook Project Briefing

> "The best documentation tells a story. This is the story of DBNotebook."

## What Are We Building Here?

Imagine you have a mountain of documents—PDFs, Word files, web pages—and you
want to have a conversation with them. Not just search, but actually ask questions
and get intelligent answers with sources. That's DBNotebook.

But here's where it gets interesting: we're not just building a simple Q&A bot.
We're building something that understands *context*, remembers your conversation,
and can even analyze Excel files to build interactive dashboards. It's like having
a research assistant who never sleeps and has read everything you've ever uploaded.

Let me show you how the magic works...
```

## Claude Code Integration
- Uses Read for deep codebase exploration
- Uses Grep/Glob for pattern discovery
- Uses Bash for git history analysis
- Uses Task for parallel exploration of large codebases
- Uses Write to create the final briefing document
- Uses WebFetch if external documentation needs to be referenced
