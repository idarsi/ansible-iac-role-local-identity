# Changelog

## Unreleased

### Breaking compatibility change

- Removed support for Debian 11 and 12, Ubuntu 24, and Enterprise Linux 8.
- Retained the supported-platform contract for Ubuntu 22 and Enterprise Linux
  9/10 (including Red Hat and Rocky normalization).
- Operators on removed platforms must remain on a previous compatible role
  revision or qualify a future support change before upgrading.
