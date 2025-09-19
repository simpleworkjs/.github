# SimpleWork.js Framework

A model-driven, real-time web application framework for Node.js that generates complete CRUD APIs and UIs from Sequelize models with automatic database migrations and live updates.

## Overview

SimpleWork.js eliminates boilerplate by automatically generating REST APIs, real-time synchronization, and declarative UIs directly from your data models. The framework is designed for small teams building internal tools, admin dashboards, and infrastructure applications where rapid development and real-time collaboration are more important than massive scale.

## Core Philosophy

- **Models are the source of truth** - All functionality derives from your Sequelize model definitions
- **Zero configuration** - Working APIs and UIs with minimal setup
- **Real-time by default** - All model changes automatically sync across connected clients
- **Metadata-driven** - Schema information powers automatic form generation and UI rendering

## Architecture

### Backend Components

1. **Model Introspection Engine** - Analyzes Sequelize models to extract schema metadata
2. **Auto-Generated REST API** - Creates full CRUD endpoints with query support
3. **Real-time Pub/Sub System** - Broadcasts model changes via WebSocket
4. **Migration Management** - Keeps database schema in sync with model definitions
5. **Authentication & Authorization** - Built-in RBAC with LDAP integration

### Frontend Components

1. **Declarative UI Renderer** - Generates forms, tables, and cards from model schemas
2. **Real-time Data Binding** - Automatic UI updates when data changes
3. **Smart Templating System** - Enhanced Mustache templates with live data
4. **Client-side Model Management** - Handles CRUD operations and state synchronization

## Getting Started

### Installation

```bash
npm install @simpleworkjs/framework
# Additional packages
npm install jq-repeat p2psub @simpleworkjs/conf
```

### Basic Setup

```javascript
const express = require('express');
const { SimpleWork } = require('@simpleworkjs/framework');

const app = express();
const sw = new SimpleWork({
  database: {
    dialect: 'sqlite',
    storage: 'database.sqlite'
  }
});

// Initialize the framework
await sw.initialize();

// Mount auto-generated routes
app.use('/api/model', sw.modelRouter);
app.use('/api', sw.authRouter);

// Start with pub/sub support
sw.startServer(app, 3000);
```

### Define Your Models

Models follow standard Sequelize patterns with additional metadata for UI generation:

```javascript
// models/user.js
module.exports = (sequelize, DataTypes, Model) => {
  class Users extends Model {
    static associate(models) {
      this.belongsToMany(models.Roles, {
        through: 'RoleUsers',
        foreignKey: 'user_id',
        as: 'roles'
      });
    }

    // Expose custom methods as API endpoints
    static exposeInstanceMethods = [
      {
        methodName: 'getRoles',
        route: 'roles',
        verb: 'get'
      }
    ];

    async getRoles() {
      // Custom business logic
      return await this.getDirectRoles();
    }
  }

  Users.init({
    username: {
      type: DataTypes.STRING,
      allowNull: false,
      primaryKey: true,
      display: {
        name: 'Username',
        description: 'Unique user identifier'
      }
    },
    email: {
      type: DataTypes.STRING,
      allowNull: false,
      display: {
        name: 'Email Address'
      }
    },
    fullName: {
      type: DataTypes.VIRTUAL,
      get() {
        return `${this.firstName} ${this.lastName}`;
      },
      display: {
        name: 'Full Name',
        title: true // Use as card/record title
      }
    },
    isActive: {
      type: DataTypes.BOOLEAN,
      defaultValue: true,
      form: {
        omit: false // Include in auto-generated forms
      },
      display: {
        name: 'Active Status'
      }
    }
  }, {
    sequelize,
    modelName: 'Users'
  });

  return Users;
};
```

## Auto-Generated APIs

The framework creates comprehensive REST endpoints for each model:

### Standard CRUD Operations

```http
GET    /api/model/Users              # List all users
POST   /api/model/Users              # Create new user
GET    /api/model/Users/:pk          # Get specific user
PUT    /api/model/Users/:pk          # Update user
DELETE /api/model/Users/:pk          # Delete user
```

### Association Management

```http
GET    /api/model/Users/:pk/roles           # Get user's roles
POST   /api/model/Users/:pk/role/:roleId    # Add role to user
DELETE /api/model/Users/:pk/role/:roleId    # Remove role from user
```

### Custom Method Endpoints

```http
GET    /api/model/Users/:pk/roles    # Custom getRoles() method
POST   /api/model/Users/syncLdap     # Static method exposure
```

### Query Parameters

```http
GET /api/model/Users?include=roles&limit=10&offset=20
GET /api/model/Users?where=isActive=true,department=IT
```

### Schema Discovery

```http
OPTIONS /api/model/Users    # Get model schema and available operations
OPTIONS /api/model/         # List all available models
```

## Declarative UI System

The frontend uses custom HTML attributes to generate interfaces automatically:

### Auto-Generated Forms

```html
<!-- Generates complete form with validation -->
<form sw-build="form" sw-model="Users"></form>

<!-- Edit existing record -->
<form sw-build="form" sw-model="Users" sw-pk="john.doe"></form>
```

### Data Tables

```html
<!-- Auto-generated table with sorting and actions -->
<div sw-build="table" sw-model="Users">
  <!-- Optional custom field template -->
  <template field="isActive">
    <span class="badge {{#isActive}}bg-success{{/isActive}}{{^isActive}}bg-danger{{/isActive}}">
      {{#isActive}}Active{{/isActive}}{{^isActive}}Inactive{{/isActive}}
    </span>
  </template>
</div>
```

### Card Layouts

```html
<!-- Card-based layout for records -->
<div sw-build="card" sw-model="Users" sw-col-size="4">
  <template field="email">
    <a href="mailto:{{email}}">{{email}}</a>
  </template>
</div>
```

### Collection Management

```html
<!-- Complete CRUD interface: form + data display -->
<div sw-build="collectionCard" sw-model="Users" sw-body-type="table"></div>
```

## Real-Time Data Binding

Use `jq-repeat` for live-updating lists:

```html
<!-- Self-updating user list -->
<div class="user-list">
  <div jq-repeat="user-list" 
       jq-for-model="Users" 
       jq-index-key="username"
       jr-order-by="fullName">
    <h4>{{fullName}}</h4>
    <p>{{email}} - {{#isActive}}Active{{/isActive}}{{^isActive}}Inactive{{/isActive}}</p>
  </div>
</div>
```

### Real-Time Features

- Automatic updates when data changes on server
- Smart DOM diffing for efficient rendering
- Sorted insertion for ordered lists
- Nested data relationships with parent-child binding

## Database Migrations

The framework includes a CLI tool for automatic migration generation:

### Generate Missing Migrations

```bash
npx simplework-migrate
```

The tool:
1. Analyzes existing migration files
2. Compares with current model definitions
3. Generates CREATE migrations for new models
4. Generates UPDATE migrations for schema changes
5. Handles both forward and rollback operations

### Example Generated Migration

```javascript
// 20241219143052-create-Users.js
"use strict";

module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.createTable("users", {
      username: {
        type: Sequelize.STRING,
        allowNull: false,
        primaryKey: true
      },
      email: {
        type: Sequelize.STRING,
        allowNull: false
      },
      createdAt: {
        allowNull: false,
        type: Sequelize.DATE
      },
      updatedAt: {
        allowNull: false,
        type: Sequelize.DATE
      }
    });
  },

  async down(queryInterface, Sequelize) {
    await queryInterface.dropTable("users");
  }
};
```

## Authentication & Authorization

Built-in RBAC system with LDAP integration:

### User Management

```javascript
// Automatic LDAP synchronization
await Users.syncLdap(); // Syncs users from LDAP server

// Role-based access control
const user = await Users.findByPk('john.doe');
const userRoles = await user.getRoles(); // Gets roles from direct assignment + groups
```

### Group & Role Management

```javascript
// Groups can be synced from LDAP
await Groups.syncLdap();

// Roles define permissions
const adminRole = await Roles.findByPk(roleId);
const roleMembers = await adminRole.getMembers(); // Direct + group members
```

## Configuration

### Database Configuration

```javascript
const sw = new SimpleWork({
  database: {
    dialect: 'postgres',
    host: 'localhost',
    database: 'myapp',
    username: 'user',
    password: 'pass'
  },
  auth: {
    ldap: {
      url: 'ldap://ldap.company.com',
      baseDN: 'dc=company,dc=com'
    }
  },
  pubsub: {
    enabled: true,
    port: 3001
  }
});
```

### Model Metadata Options

```javascript
// Field-level configuration
{
  type: DataTypes.STRING,
  display: {
    name: 'Display Name',      // UI label
    description: 'Help text',  // Form help text
    omit: false,              // Hide from displays
    title: true               // Use as record title
  },
  form: {
    omit: true                // Hide from forms
  },
  references: {
    model: 'OtherModel',      // For dropdowns
    key: 'id'
  }
}
```

## Pub/Sub Events

The framework automatically publishes model events:

```javascript
// Subscribe to specific model changes
app.subscribe('model:Users:create', (data, topic) => {
  console.log('New user created:', data.instance);
});

// Subscribe to all events for a model
app.subscribe(/^model:Users:/, (data, topic) => {
  console.log('User event:', topic, data);
});
```

## Best Practices

### Model Design

1. Use descriptive `display` metadata for better UIs
2. Define `virtual` fields for computed values
3. Expose business logic methods via `exposeInstanceMethods`
4. Use appropriate data types for automatic form generation

### Real-Time Considerations

1. Use `jq-index-key` for stable list ordering
2. Implement custom `parseKeys` for data formatting
3. Consider throttling for high-frequency updates

### Performance

1. Use `include` parameters to avoid N+1 queries
2. Implement pagination with `limit`/`offset`
3. Add database indexes for frequently queried fields

## Limitations

- Designed for applications with < 50 concurrent users
- Real-time features require WebSocket support
- Complex business logic may require custom endpoints
- UI customization has boundaries - not a replacement for custom frontends

## Use Cases

Ideal for:
- Internal admin dashboards
- Infrastructure monitoring tools
- Lab/research data management
- Configuration management interfaces
- Rapid prototyping of data-driven applications

Not suitable for:
- Public-facing applications requiring custom UX
- High-traffic consumer applications
- Applications with complex, non-CRUD workflows
- Mobile-first applications

## Contributing

The framework is built from real-world usage in infrastructure and internal tooling. Contributions should focus on maintaining simplicity while adding practical functionality for small-team development scenarios.
