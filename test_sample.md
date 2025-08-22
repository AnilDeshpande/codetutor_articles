# Security Documentation

This document provides security-related information for the CodeTutor web application.

## Recent Security Fixes

- **[Infinite Request Loop Fix](./INFINITE_REQUEST_LOOP_FIX.md)** (August 22, 2025): Resolved authentication system infinite loops and profile page access issues while maintaining security measures.

## Authorization Framework

The CodeTutor application implements a role-based access control (RBAC) system with the following components:

1. **Roles**: Each user is assigned one or more roles (admin, premium-user, basic-user, etc.)
2. **Permissions**: Each role has a set of permissions in the format `resource:action`
3. **Middleware**: The `can()` middleware checks if a user has permission for an action on a resource

## Securing Sensitive Endpoints

### MCP (Model Context Protocol) Endpoint

The MCP endpoint allows natural language queries to MongoDB collections. It is secured with:

- JWT authentication
- Role-based permission checks (`mcp:query` permission)
- Collection-specific permission checks
- Input validation
- Server-to-server authentication

For details, see the [MCP Security Enhancements](./packages/backend/docs/McpSecurityEnhancements.md) documentation.

### Admin Endpoints

Admin endpoints are protected with the `hasRole('admin')` middleware, which ensures only users with the admin role can access them.

## Security Scripts

The following scripts can be used to manage security:

- `setupMcpPermissions.js`: Configure MCP permissions to ensure only admins can access
- `testMcpAuthorization.js`: Test MCP endpoint authorization
- `rotate_secrets.sh`: Rotate JWT secrets and other sensitive credentials

## Best Practices

1. Always use the `can()` middleware for new endpoints that require specific permissions
2. Log security-relevant events for audit purposes
3. Validate all input parameters
4. Use the principle of least privilege when assigning permissions to roles
