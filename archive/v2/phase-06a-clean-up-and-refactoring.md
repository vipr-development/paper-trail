Write prompt for:

- Restructuring to extract common analysis metrics
- Identify opportunities for performance tweaking
- Separating react stuff from common stuff
- Making a base formatter and making that the base for the overview formatter (and all formatters). Currently the overview formatter is the base
- This probably needs to be a monorepo
  - Core analyzer engine
  - React-specific analyzer stuff
  - VSCode extension
  - CLI
- This _could_ become Vipr by extension, starting with React and expanding out.
