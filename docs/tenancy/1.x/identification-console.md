---
title: Console Identification
icon: fal fa-terminal
excerpt: >
    Identifies tenants using the console/artisan.
tags:
    - identification
    - console
    - driver
    - artisan
---

# Tenant Identification: Console

- [Overview](#overview)
- [Introduction](#introduction)
- [Installation](#installation)
- [Configuration](#configuration)
  - [Example](#example)
  - [Advance Example](#advance-example)
- [Usage](#usage)

## Overview

**Purpose**

The purpose of this package is to allow a [Tenant](tenant-what-is) to be identified though console commands

**Requirements**

The Tenant [must be registered in the TenantResolver](tenant-setup)

**Use Cases**

- Identify a Tenant by a custom field
- Identify a Tenant by an ID
- And many more!

## Introduction

Uses the console input object to identify a tenant. 

In order to allow a tenant to be identified with the Console driver, all
commands have gained two new options, `--tenant` and `--tenant-identifier`.

For a tenant to be identified with the Console driver, you
need to apply a Contract to the [tenant][tenant-what-is] class and implement the required
methods.

## Installation

> Make sure that the model you are using [is registered in the TenantResolver](identification-general).

### Using Tenancy/Framework
Install via composer:
```bash
composer require tenancy/identification-driver-console
```

### Using Tenancy/Tenancy or with provider discovery disabled
Register the following ServiceProvider: 
  - `Tenancy\Identification\Drivers\Console\Providers\IdentificationProvider::class`

## Configuration

Tenants that will be identified by console need to implement the `Tenancy\Identification\Drivers\Console\Contracts\IdentifiesByConsole` contract.

The `tenantIdentificationByConsole` method should return the tenant that is identified based on the current request.

### Example

The following example assumes your Customer has a slug column with which we use to identify it.

```php
use Illuminate\Database\Eloquent\Model;
use Tenancy\Identification\Concerns\AllowsTenantIdentification;
use Tenancy\Identification\Contracts\Tenant;
use Tenancy\Identification\Drivers\Console\Contracts\IdentifiesByConsole;
use Symfony\Component\Console\Input\InputInterface;

class Customer extends Model implements Tenant, IdentifiesByConsole
{
    use AllowsTenantIdentification;
  
    /**
     * Specify whether the tenant model is matching the request.
     *
     * @param Request $request
     * @return Tenant
     */
    public function tenantIdentificationByConsole(InputInterface $input): ?Tenant
    {
        if ($input->hasParameterOption('--tenant')) {
            return $this->query()
                ->where('slug', $input->getParameterOption('--tenant'))
                ->first();
        }
        
        return null;
    }
}
```

### Advance Example

In the following example, we will expand upon the previous example with the assumption that you have multiple Tenant types, that may share the same slug.

```php
use Illuminate\Database\Eloquent\Model;
use Tenancy\Identification\Concerns\AllowsTenantIdentification;
use Tenancy\Identification\Contracts\Tenant;
use Tenancy\Identification\Drivers\Console\Contracts\IdentifiesByConsole;
use Symfony\Component\Console\Input\InputInterface;

class Customer extends Model implements Tenant, IdentifiesByConsole
{
    use AllowsTenantIdentification;
  
    /**
     * Specify whether the tenant model is matching the request.
     *
     * @param Request $request
     * @return Tenant|null
     */
    public function tenantIdentificationByConsole(InputInterface $input): ?Tenant
    {
        if ($input->hasParameterOption('--tenant-identifier')) {
            if($input->getParameterOption('--tenant-identifier') != $this->getTenantIdentifier()) {
                return null;
            }
        }
        if ($input->hasParameterOption('--tenant')) {
            return $this->query()
                ->where('slug', $input->getParameterOption('--tenant'))
                ->first();
        }
        
        return null;
    }
}
```



## Usage

### `--tenant`

This option allows you to specify a specific Tenant.

In the following example we will assume that you want to list the routes available for a specific tenant.

We will also assume that the tenant in question has the previous example implemented, has the slug of "my-first-tenant".

```bash
php artisan route:list --tenant=my-first-tenant
```

### `--tenant-identifier`

This option allows you to specify a specific Tenant type.

Because Tenants can be multiple classes this option is useful when writing custom commands to do something specific to different types of Tenants, or when different types of tenants could be identified by the same `--tenant` value.

#### Custom Command

In the following example command, we will assume that every tenant has a "maintenance" value to enable or disable it from being redirected to a maintenance page without having to bring down the entire application.

We will also assume that you have two Tenants; User's and Organization's. 

```php
namespace App\Console\Commands;

use Illuminate\Console\Command;

class TenantMaintenanceCommand extends Command
{
    protected $signature = 'tenant:maintenance';

    protected $description = 'Inverts the Maintenance mode for a tenant';

    /** @var Resolver */
    protected $resolver;

    public function __construct(ResolvesTenants $resolver)
    {
        parent::__construct();
        $this->resolver = $resolver;
    }

    public function handle()
    {
        $this->resolver->getModels()->each(function (string $class) {
            $model = (new $class());

            if ($this->option('tenant-identifier') != $model->getTenantIdentifier()) {
                return;
            }

            $model->all()->each(function($tenant) {
                $tenant->maintenance = !$tenant->maintenance;
                $tenant->save();
            })

        });
    }
}
```

#### Usage

In order to run your custom command only for User Tenants you can do the following:

```bash
php artisan tenant:maintenance --tenant-identifier=mysql.users
```

