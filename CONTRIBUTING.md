# Contributing to my Ansible playbook

Thank you for your interest in contributing to this Ansible playbook! Here are some guidelines to help you get started.

## Getting Started

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Make your changes
4. Commit your changes (`git commit -m 'Add some amazing feature'`)
5. Push to the branch (`git push origin feature/amazing-feature`)
6. Open a Pull Request

## Project Structure

Before contributing, please familiarize yourself with the project structure:

- `roles/` - Contains individual service roles
- `group_vars/` - Variable definitions for host groups
- `inventory/` - Host definitions and groupings
- `site.yml` - Main playbook

## Guidelines

### Adding a New Role

1. Create a directory in `roles/` with your role name
2. Include at minimum:
   - `tasks/main.yml` - Main tasks for your role
   - `defaults/main.yml` - Default variables
   - `handlers/main.yml` - Any handlers your role needs
   - `templates/` - Templates your role uses
3. Document the role in the README.md

### Security Considerations

- Never commit real credentials or sensitive information
- Use the existing pattern for managing secrets
- Make sure example files don't contain sensitive defaults

### Testing

- Test your changes on a non-production environment first
- Verify idempotence (running the playbook twice should not cause changes the second time)
- Ensure your role works with the base role

## Code Style

- Follow YAML best practices
- Add comments for complex operations
- Use clear, descriptive variable and task names

## Documentation

- Update relevant documentation when adding features
- Add descriptions for new variables
- Document any new requirements

Thank you for contributing!
