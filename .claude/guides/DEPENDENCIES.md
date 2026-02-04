# Dependency Management Guide

## Package Manager: Bun First, Node.js Fallback

- **Default:** Use Bun for package management (`bun install`, `bun add`, `bun remove`)
- **Fallback:** Use npm/Node.js only if Bun is incompatible with specific packages
- **Lock Files:** Commit `bun.lockb` or `package-lock.json` to version control

---

## Adding Dependencies

### STOP and ask Ozgur before:
- Adding new dependencies (justify the need, explore alternatives)
- Upgrading major versions (breaking changes require discussion)
- Adding packages that significantly increase bundle size

### Proceed without asking for:
- Patch/minor version updates for security fixes
- Dev dependencies that don't affect production bundle
- Type definition packages (`@types/*`)

---

## Updating Dependencies

```bash
# Check for outdated packages
bun outdated

# Update specific package (ask first for major versions)
bun update <package-name>

# Security updates (proceed immediately)
bun update <package-name> --latest
```

---

## Dependency Conflicts

1. **Check compatibility:** Review package documentation for version requirements
2. **Use resolution field:** Add to `package.json` if needed (ask Ozgur first)
3. **Document rationale:** Comment why specific versions are pinned
4. **Never force install:** If dependencies conflict, investigate root cause

**Example resolution:**
```json
{
  "resolutions": {
    "package-name": "1.2.3"
  }
}
```
