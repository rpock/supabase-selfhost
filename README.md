# Supabase Self-Hosting Resources

This repository contains various resources, guides, and configuration snippets for self-hosted Supabase instances. These materials are collected from practical experiences and might be helpful for others running their own Supabase deployments.

## Available Resources

### [PostgreSQL SSL Certificate Auto-Renewal](postgres-ssl-certificate.md)

A complete guide for implementing automatic SSL certificate renewal for Supabase Postgres databases using nginx-proxy-manager. This solution works even in environments where direct modification of postgresql.conf is not possible. It includes:

- A Docker Compose configuration
- A custom wrapper script that monitors and updates SSL certificates 
- Detailed explanations of how the system works
- Best practices for certificate handling

## Purpose

This repository aims to share practical solutions for common challenges in self-hosting Supabase. These resources supplement the official documentation with real-world implementations that solve specific problems.

New content will be added periodically as more solutions are developed and tested.

## Contributions

Feel free to suggest improvements or share your own solutions for self-hosted Supabase instances by opening an issue or pull request.

## License

All content in this repository is shared under the MIT License unless otherwise specified.
