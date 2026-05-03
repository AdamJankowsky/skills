# AI-Assisted Development Skill

## Overview

This skill enables an iterative, git-based AI-assisted development workflow where the developer owns architecture and pseudocode, while the AI handles implementation, boilerplate, and testing. The workflow is IDE-agnostic and relies on git for state management and special comment markers for communication.

## Core Philosophy

- **Developer owns design**: Developer writes structure, pseudocode, and key logic patterns
- **AI owns implementation**: AI converts pseudocode to working code, writes boilerplate, and generates tests
- **Git as state machine**: Each phase of development is tracked through git commits and branches
- **Explicit communication**: All AI-human interaction happens through structured comments in code

## Workflow Flow

```
[Dev] Planning Chat
    ↓
[Dev] Write Markers/Pseudocode (uncommitted)
    ↓
[Dev] "Ready for implementation"
    ↓
[AI] Review git diff → Implement → Add questions if needed
    ↓
[Has questions?]
    ├─ Yes → [AI] Add AI:ASK comments → [Dev] Review/Answer → Back to AI Review
    └─ No → [Dev] Review changes
              ↓
    [Dev satisfied?]
    ├─ No → [Dev] Add new markers/modify code → Back to AI Review
    └─ Yes → Done (no more AI/DEV comments)
```

**Success Condition**: Implementation is complete when there are no more `AI:ASK`, `AI:IMPL`, or `PSEUDO:` comments in the code, OR the developer explicitly states "Looks good" / "Implementation complete".

## Workflow Phases

### Phase 1: Planning
**Trigger**: Developer starts session and describes feature requirements in chat.

1. AI analyzes requirements and identifies files to create/modify
2. AI outputs a structured plan:
   ```
   Files to create:
   - src/services/emailValidator.ts (new utility)
   
   Files to modify:
   - src/routes/user.ts (add validation endpoint)
   - src/models/user.ts (add email field)
   ```
3. AI waits for developer confirmation or asks clarifying questions

### Phase 2: Development (Human)
**Trigger**: Developer tells AI "I'm ready to write pseudocode" or similar.

1. AI creates empty files/functions with minimal scaffolding if needed
2. Developer writes code, pseudocode, and markers using the comment syntax below in the working tree (uncommitted)
3. Developer does NOT commit - markers stay in uncommitted changes
4. Developer signals completion by telling AI in chat: "Ready for implementation"

### Phase 3: AI Review & Implementation
**Trigger**: Developer tells AI to implement in chat.

1. AI runs `git diff` to see all uncommitted changes in the working tree
2. AI identifies all marker comments (`AI:IMPL`, `PSEUDO`, `AI:ASK`) from the diff
3. AI implements missing methods, converts pseudocode to actual code
4. AI fixes compilation errors encountered during implementation
5. AI writes tests to cover implemented functionality
6. **If AI has questions**: AI adds `// AI:ASK:` comments instead of guessing, then commits as `[AI-REVIEW] Questions about implementation`
7. **If no questions**: AI cleans up all marker comments and commits as `[AI-IMPL] Implemented feature`

### Phase 4: Developer Review & Iteration
**Trigger**: AI tells developer "Implementation ready for review" OR developer sees `AI:ASK` comments.

**Path A - AI asked questions**:
1. Developer sees `// AI:ASK:` comments in code
2. Developer answers by either:
   - Modifying code directly with clarifications (uncommitted)
   - Responding in chat
3. Developer tells AI: "Continue implementation"
4. Back to **Phase 3** - AI reviews diff, implements based on answers, removes answered questions

**Path B - Developer wants changes**:
1. Developer reviews implemented code
2. If changes needed, developer adds new markers/pseudocode (uncommitted)
3. Developer tells AI: "Please update implementation"
4. Back to **Phase 3** - AI reviews diff, implements changes

**Path C - Implementation complete**:
1. Developer reviews code, everything looks correct
2. Developer tells AI: "Looks good" or "Implementation complete"
3. AI does final check - if no markers remain, marks implementation as successful
4. Feature is done

## Comment Syntax

All markers are language-agnostic. Use the comment style appropriate for your language (`//`, `#`, `<!--`, etc.).

### Developer → AI Markers

```typescript
// AI:IMPL function_name
// Brief description of what to implement
function function_name() {
    // AI will fill this in
}

// AI:READ
// Tells AI to read this file for context but not modify it

// PSEUDO: Validate email format
// 1. Check if string contains exactly one @
// 2. Split into local and domain parts
// 3. Check domain has at least one dot
// 4. Return true/false
function validateEmail(email: string): boolean {
    // AI will replace pseudocode with actual implementation
}
```

**Rules:**
- `AI:IMPL` marks a function/section AI should implement
- `PSEUDO:` marks a block of pseudocode AI should convert to real code
- `AI:READ` tells AI to read a file for context without modifying it
- Both can coexist - pseudocode above a function guides the implementation
- Developer can write partial code; AI preserves what exists and fills gaps

### AI → Developer Markers

```typescript
// AI:ASK: [category]: [question]?
// Example:
// AI:ASK: validation: Should empty strings be rejected or treated as null?
```

**Rules:**
- AI adds this when it needs clarification during implementation
- Format: `AI:ASK:` followed by category, colon, then the question
- Developer answers by modifying code with clarifications or responding in chat
- After implementation round, AI removes all answered `AI:ASK` markers

### Context Markers

```typescript
// AI:CTX: [information for AI]
// Example:
// AI:CTX: This function must be async because it calls an external API
```

**Rules:**
- `AI:CTX` provides context to help AI make better decisions
- These are optional and may be kept or removed at developer's discretion
- Not automatically cleaned up by AI

## Git Workflow

### Branch Naming
- `feature/description` - Main implementation branch
- No strict naming required, but consistent prefixes help

### Commit Messages
AI commits during implementation:
```
[AI-IMPL] Implement email validation functions
[AI-TEST] Add tests for email validation
[AI-FIX] Resolve compilation errors in validator
[AI-REVIEW] Questions about implementation (when AI has AI:ASK comments)
```

### Diff Strategy
AI uses `git diff` (showing uncommitted working tree changes) to see exactly what the developer has written in the current iteration. This provides the precise context needed - only modified/added files with developer markers are considered for implementation. No need to re-read unchanged files.

## Implementation Rules for AI

1. **Preserve developer code**: Never overwrite logic the developer has written unless explicitly marked for replacement
2. **Pseudocode conversion**: Replace `PSEUDO:` blocks with working code that matches the described algorithm
3. **Error fixing**: Compilation errors in pseudocode are fixed as part of implementation
4. **Test generation**: Write tests after implementation that cover all implemented paths
5. **Comment cleanup**: Remove all `AI:IMPL`, `PSEUDO:`, and answered `AI:ASK` markers after implementation
6. **Style consistency**: Match existing code style, formatting, and patterns in the codebase
7. **Minimal surprise**: If the implementation differs significantly from pseudocode intent, add an `AI:ASK` instead of guessing
8. **Success check**: Before finishing, verify no `AI:ASK`, `AI:IMPL`, or `PSEUDO:` markers remain (unless developer explicitly said "Looks good")

## Example Session

### Planning
**Developer**: "I need to add email validation to the user registration"

**AI**: 
```
Files to create:
- src/utils/emailValidator.ts

Files to modify:
- src/routes/register.ts (add validation call)
- src/models/user.ts (no changes needed, validation is external)

Plan:
1. Create emailValidator.ts with validateEmail() function
2. Update register route to call validator before saving user
```

**Developer**: "Looks good, I'll write the pseudocode"

### Development
**Developer** creates `src/utils/emailValidator.ts`:
```typescript
// AI:IMPL validateEmail
// PSEUDO: Check email format
// 1. Must contain exactly one @
// 2. Local part (before @) must not be empty
// 3. Domain part (after @) must contain at least one dot
// 4. No spaces allowed
export function validateEmail(email: string): boolean {
    // AI will implement
}

// AI:IMPL getValidationError
// Return human-readable error message for invalid emails
export function getValidationError(email: string): string {
    // AI will implement
}
```

**Developer** modifies `src/routes/register.ts`:
```typescript
import { validateEmail, getValidationError } from '../utils/emailValidator';

// In registration handler:
// AI:IMPL email validation check
if (!validateEmail(email)) {
    // return 400 with error message from getValidationError
}
```

**Developer** does NOT commit - markers stay in working tree as uncommitted changes

**Developer** in chat: "Ready for implementation"

### Implementation Round 1
**AI** runs `git diff`, sees the uncommitted markers, implements functions, adds tests:

`src/utils/emailValidator.ts`:
```typescript
export function validateEmail(email: string): boolean {
    if (!email || typeof email !== 'string') {
        return false;
    }
    const atCount = email.split('@').length - 1;
    if (atCount !== 1) return false;
    
    const [local, domain] = email.split('@');
    if (!local || local.length === 0) return false;
    if (!domain || !domain.includes('.')) return false;
    if (email.includes(' ')) return false;
    
    return true;
}

export function getValidationError(email: string): string {
    if (!email) return 'Email is required';
    if (!email.includes('@')) return 'Email must contain @';
    if (email.split('@').length - 1 > 1) return 'Email must contain only one @';
    const [, domain] = email.split('@');
    if (!domain || !domain.includes('.')) return 'Email domain must contain a dot';
    if (email.includes(' ')) return 'Email cannot contain spaces';
    return 'Invalid email format';
}
```

`src/utils/emailValidator.test.ts` (new file):
```typescript
import { validateEmail, getValidationError } from './emailValidator';

describe('validateEmail', () => {
    test('returns true for valid email', () => {
        expect(validateEmail('user@example.com')).toBe(true);
    });
    
    test('returns false for missing @', () => {
        expect(validateEmail('userexample.com')).toBe(false);
    });
    
    test('returns false for multiple @', () => {
        expect(validateEmail('user@@example.com')).toBe(false);
    });
    
    test('returns false for empty local part', () => {
        expect(validateEmail('@example.com')).toBe(false);
    });
    
    test('returns false for missing domain dot', () => {
        expect(validateEmail('user@example')).toBe(false);
    });
    
    test('returns false for spaces', () => {
        expect(validateEmail('user @example.com')).toBe(false);
    });
});
```

**AI** commits: `[AI-IMPL] Implement email validation and tests`

### Review Round 1
**Developer** reviews code:

**Developer**: "Looks good but I need to handle international emails too. Let me add some pseudocode for that."

### Development Round 2
**Developer** modifies `src/utils/emailValidator.ts` (uncommitted):
```typescript
// PSEUDO: Add international email support
// Allow Unicode characters in local part
// Check for punycode domains (xn--)
// Keep existing ASCII validation as fallback
```

**Developer** in chat: "Please update with international support"

### Implementation Round 2
**AI** runs `git diff`, sees new pseudocode, implements:

```typescript
export function validateEmail(email: string): boolean {
    if (!email || typeof email !== 'string') {
        return false;
    }
    
    // Check for international characters
    const hasUnicode = /[^\x00-\x7F]/.test(email);
    
    const atCount = email.split('@').length - 1;
    if (atCount !== 1) return false;
    
    const [local, domain] = email.split('@');
    if (!local || local.length === 0) return false;
    
    // Allow punycode domains (internationalized domain names)
    if (!domain) return false;
    const isValidDomain = domain.includes('.') || domain.startsWith('xn--');
    if (!isValidDomain) return false;
    
    if (email.includes(' ')) return false;
    
    return true;
}
```

**AI** commits: `[AI-IMPL] Add international email support`

### Review Round 2 - Success
**Developer**: "Looks good, implementation complete"

**AI** verifies no markers remain → marks as successful

## Success Criteria

Implementation is marked as successful when ANY of the following is true:

1. **No markers remain**: No `AI:ASK`, `AI:IMPL`, or `PSEUDO:` comments exist in the codebase
2. **Developer approval**: Developer explicitly says "Looks good", "Implementation complete", or "Done"
3. **No uncommitted changes**: Working tree is clean after AI implementation (all markers processed)

## Best Practices

### For Developers
- Write pseudocode that describes intent, not implementation details
- Mark only the parts you want AI to implement
- Review AI changes before approving
- Keep iterations small and focused
- Use `AI:READ` to give AI context from existing files without modifying them

### For AI
- Always run `git diff` first to see uncommitted changes from developer
- Process only markers found in the working tree diff
- Never assume - ask with `AI:ASK` when intent is unclear
- Preserve all non-marked developer code exactly
- Generate tests that cover edge cases from pseudocode
- Clean up all markers after successful implementation
- Match existing project patterns and conventions
- Before finishing, verify success criteria are met

## Language Support

This skill works with any language. Comment markers adapt to language syntax:

- **JavaScript/TypeScript/C/Java/Go**: `// AI:IMPL`, `// PSEUDO:`, `// AI:ASK:`, `// AI:READ`, `// AI:CTX`
- **Python/Ruby/Shell**: `# AI:IMPL`, `# PSEUDO:`, `# AI:ASK:`, `# AI:READ`, `# AI:CTX`
- **HTML/XML**: `<!-- AI:IMPL -->`, `<!-- PSEUDO: -->`, `<!-- AI:ASK: -->`, `<!-- AI:READ -->`, `<!-- AI:CTX -->`
- **CSS**: `/* AI:IMPL */`, `/* PSEUDO: */`, `/* AI:ASK: */`, `/* AI:READ */`, `/* AI:CTX */`

## Troubleshooting

**AI implements wrong thing**: Add more specific pseudocode or use `AI:CTX` markers to clarify intent.

**Tests fail**: Check if pseudocode described edge cases. AI generates tests from pseudocode descriptions.

**Compilation errors persist**: AI should fix these, but if they require architectural changes, add `AI:ASK` and discuss.

**Too much back-and-forth**: In planning phase, be more specific about requirements. Break large features into smaller chunks.

**AI asks too many questions**: Provide more context in `AI:CTX` markers or write more detailed pseudocode.

## Summary

This skill creates a structured collaboration where:
- **Developer** = Architect + Product Owner (designs, decides, reviews, approves)
- **AI** = Senior Developer (implements, tests, asks clarifying questions)
- **Git** = State management (tracks phases, enables diff-based context)
- **Comments** = Communication protocol (explicit, versioned, cleanup-friendly)

The workflow scales from small features (single file) to large refactors (many files) while maintaining developer control and AI efficiency. The iterative cycle ensures quality through continuous review and clarification.
