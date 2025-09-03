# Contributing Guide

## Welcome Contributors!

We welcome contributions to the Grax Data Lake Knowledge project. This guide will help you get started.

## Development Setup

### Prerequisites

- Git
- Node.js 18+ or Python 3.9+
- Docker and Docker Compose
- Your favorite code editor

### Local Development Environment

1. Fork and clone the repository:
   ```bash
   git clone https://github.com/YOUR_USERNAME/grax-datalake-knowledge.git
   cd grax-datalake-knowledge
   ```

2. Install dependencies:
   ```bash
   npm install
   # or
   pip install -r requirements-dev.txt
   ```

3. Set up pre-commit hooks:
   ```bash
   npm run prepare
   # or
   pre-commit install
   ```

4. Start development environment:
   ```bash
   docker-compose -f docker-compose.dev.yml up -d
   npm run dev
   ```

## Contribution Process

### 1. Issue First

Before starting work:
- Check existing issues
- Create a new issue if needed
- Discuss your approach
- Get feedback from maintainers

### 2. Branch Strategy

- `main`: Production-ready code
- `develop`: Integration branch
- `feature/feature-name`: New features
- `bugfix/issue-description`: Bug fixes
- `hotfix/critical-fix`: Critical production fixes

### 3. Making Changes

1. Create a feature branch:
   ```bash
   git checkout -b feature/your-feature-name
   ```

2. Make your changes following coding standards

3. Write or update tests:
   ```bash
   npm test
   # or
   pytest
   ```

4. Update documentation if needed

5. Commit with conventional commit messages:
   ```bash
   git commit -m "feat: add new data processing pipeline"
   ```

### 4. Pull Request Process

1. Push your branch:
   ```bash
   git push origin feature/your-feature-name
   ```

2. Create a Pull Request with:
   - Clear title and description
   - Link to related issues
   - Screenshots if applicable
   - Test results

3. Address review feedback

4. Ensure CI passes

## Coding Standards

### Code Style

- **JavaScript/TypeScript**: ESLint + Prettier
- **Python**: Black + Flake8
- **Documentation**: Markdown with consistent formatting

### Commit Messages

Use Conventional Commits format:

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

Types:
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes
- `refactor`: Code refactoring
- `test`: Test changes
- `chore`: Maintenance tasks

### Testing

- Write unit tests for new features
- Maintain test coverage above 80%
- Include integration tests for APIs
- Test edge cases and error conditions

```bash
# Run tests
npm test
npm run test:coverage

# or for Python
pytest
pytest --cov=src
```

## Documentation

### Types of Documentation

1. **Code Documentation**: Inline comments and docstrings
2. **API Documentation**: OpenAPI/Swagger specs
3. **User Documentation**: Guides and tutorials
4. **Developer Documentation**: Architecture and design docs

### Documentation Standards

- Use clear, concise language
- Include code examples
- Keep documentation up-to-date
- Use consistent formatting

## Review Process

### What We Look For

- **Functionality**: Does it work as intended?
- **Code Quality**: Is it readable and maintainable?
- **Tests**: Are there adequate tests?
- **Documentation**: Is it properly documented?
- **Performance**: Does it impact performance?
- **Security**: Are there security considerations?

### Review Timeline

- Initial response: Within 2 business days
- Full review: Within 1 week
- Follow-up reviews: Within 2 business days

## Getting Help

### Communication Channels

- **GitHub Issues**: Bug reports and feature requests
- **GitHub Discussions**: General questions and discussions
- **Slack**: Real-time communication (invite required)
- **Email**: Direct contact with maintainers

### Resources

- [Architecture Documentation](../architecture/overview.md)
- [API Documentation](../api/README.md)
- [Setup Guide](../setup/installation.md)

## Recognition

We recognize contributors through:

- Contributor list in README
- Release notes mentions
- GitHub contributor stats
- Special recognition for significant contributions

## Code of Conduct

We follow the [Contributor Covenant](https://www.contributor-covenant.org/version/2/1/code_of_conduct/).

Be respectful, inclusive, and professional in all interactions.

## License

By contributing, you agree that your contributions will be licensed under the same license as the project.
