---
name: inertia-svelte-development
description: "Develops Inertia.js v2 Svelte client-side applications. Activates when creating Svelte pages, forms, or navigation; using Link, Form, or router; working with deferred props, prefetching, or polling; or when user mentions Svelte with Inertia, Svelte pages, Svelte forms, or Svelte navigation."
license: MIT
metadata:
  author: laravel
---

# Inertia Svelte Development

## When to Apply

Activate this skill when:

- Creating or modifying Svelte page components for Inertia
- Working with forms in Svelte (using `<Form>` or `useForm`)
- Implementing client-side navigation with `<Link>` or `router`
- Using v2 features: deferred props, prefetching, WhenVisible, InfiniteScroll, once props, flash data, or polling
- Building Svelte-specific features with the Inertia protocol

## Documentation

Use `search-docs` for detailed Inertia v2 Svelte patterns and documentation.

## Basic Usage

### Page Components Location

Svelte page components should be placed in the `resources/js/pages` directory.

### Page Component Structure

<!-- Basic Svelte Page Component -->
```svelte
<script>
export let users
</script>

<div>
    <h1>Users</h1>
    <ul>
        {#each users as user (user.id)}
            <li>{user.name}</li>
        {/each}
    </ul>
</div>
```

## Client-Side Navigation

### Basic Link Component

Use `<Link>` for client-side navigation instead of traditional `<a>` tags:

<!-- Inertia Svelte Navigation -->
```svelte
<script>
import { Link } from '@inertiajs/svelte'
</script>

<Link href="/">Home</Link>
<Link href="/users">Users</Link>
<Link href={`/users/${user.id}`}>View User</Link>
```

### Link With Method

<!-- Link With POST Method -->
```svelte
<script>
import { Link } from '@inertiajs/svelte'
</script>

<Link href="/logout" method="post">Logout</Link>
```

### Prefetching

Prefetch pages to improve perceived performance:

<!-- Prefetch on Hover -->
```svelte
<script>
import { Link } from '@inertiajs/svelte'
</script>

<Link href="/users" prefetch>Users</Link>
```

### Programmatic Navigation

<!-- Router Visit -->
```svelte
<script>
import { router } from '@inertiajs/svelte'

function handleClick() {
    router.visit('/users')
}

// Or with options
function createUser() {
    router.visit('/users', {
        method: 'post',
        data: { name: 'John' },
        onSuccess: () => console.log('Success!'),
    })
}
</script>
```

## Form Handling

### Form Component (Recommended)

The recommended way to build forms is with the `<Form>` component:

<!-- Form Component Example -->
```svelte
<script>
import { Form } from '@inertiajs/svelte'
</script>

<Form action="/users" method="post" let:errors let:processing let:wasSuccessful>
    <input type="text" name="name" />
    {#if errors.name}
        <div>{errors.name}</div>
    {/if}

    <input type="email" name="email" />
    {#if errors.email}
        <div>{errors.email}</div>
    {/if}

    <button type="submit" disabled={processing}>
        {processing ? 'Creating...' : 'Create User'}
    </button>

    {#if wasSuccessful}
        <div>User created!</div>
    {/if}
</Form>
```

### Form Component With All Props

<!-- Form Component Full Example -->
```svelte
<script>
import { Form } from '@inertiajs/svelte'
</script>

<Form
    action="/users"
    method="post"
    let:errors
    let:hasErrors
    let:processing
    let:progress
    let:wasSuccessful
    let:recentlySuccessful
    let:clearErrors
    let:resetAndClearErrors
    let:defaults
    let:isDirty
    let:reset
    let:submit
>
    <input type="text" name="name" value={defaults.name} />
    {#if errors.name}
        <div>{errors.name}</div>
    {/if}

    <button type="submit" disabled={processing}>
        {processing ? 'Saving...' : 'Save'}
    </button>

    {#if progress}
        <progress value={progress.percentage} max="100">
            {progress.percentage}%
        </progress>
    {/if}

    {#if wasSuccessful}
        <div>Saved!</div>
    {/if}
</Form>
```

### Form Component Reset Props

The `<Form>` component supports automatic resetting:

- `resetOnError` - Reset form data when the request fails
- `resetOnSuccess` - Reset form data when the request succeeds
- `setDefaultsOnSuccess` - Update default values on success

Use the `search-docs` tool with a query of `form component resetting` for detailed guidance.

<!-- Form With Reset Props -->
```svelte
<script>
import { Form } from '@inertiajs/svelte'
</script>

<Form
    action="/users"
    method="post"
    resetOnSuccess
    setDefaultsOnSuccess
    let:errors
    let:processing
    let:wasSuccessful
>
    <input type="text" name="name" />
    {#if errors.name}
        <div>{errors.name}</div>
    {/if}

    <button type="submit" disabled={processing}>
        Submit
    </button>
</Form>
```

Forms can also be built using the `useForm` hook for more programmatic control. Use the `search-docs` tool with a query of `useForm helper` for guidance.

### `useForm` Hook

For more programmatic control or to follow existing conventions, use the `useForm` hook:

<!-- useForm Example -->
```svelte
<script>
import { useForm } from '@inertiajs/svelte'

const form = useForm({
    name: '',
    email: '',
    password: '',
})

function submit() {
    $form.post('/users', {
        onSuccess: () => $form.reset('password'),
    })
}
</script>

<form on:submit|preventDefault={submit}>
    <input type="text" bind:value={$form.name} />
    {#if $form.errors.name}
        <div>{$form.errors.name}</div>
    {/if}

    <input type="email" bind:value={$form.email} />
    {#if $form.errors.email}
        <div>{$form.errors.email}</div>
    {/if}

    <input type="password" bind:value={$form.password} />
    {#if $form.errors.password}
        <div>{$form.errors.password}</div>
    {/if}

    <button type="submit" disabled={$form.processing}>
        Create User
    </button>
</form>
```

## Inertia v2 Features

### Deferred Props

Use deferred props to load data after initial page render:

<!-- Deferred Props with Empty State -->
```svelte
<script>
export let users
</script>

<div>
    <h1>Users</h1>
    {#if !users}
        <div class="animate-pulse">
            <div class="h-4 bg-gray-200 rounded w-3/4 mb-2"></div>
            <div class="h-4 bg-gray-200 rounded w-1/2"></div>
        </div>
    {:else}
        <ul>
            {#each users as user (user.id)}
                <li>{user.name}</li>
            {/each}
        </ul>
    {/if}
</div>
```

### Polling

Automatically refresh data at intervals:

<!-- Polling Example -->
```svelte
<script>
import { router } from '@inertiajs/svelte'
import { onMount, onDestroy } from 'svelte'

export let stats

let interval

onMount(() => {
    interval = setInterval(() => {
        router.reload({ only: ['stats'] })
    }, 5000) // Poll every 5 seconds
})

onDestroy(() => {
    clearInterval(interval)
})
</script>

<div>
    <h1>Dashboard</h1>
    <div>Active Users: {stats.activeUsers}</div>
</div>
```

### WhenVisible

Lazy-load a prop when an element scrolls into view. Useful for deferring expensive data that sits below the fold:

<!-- WhenVisible Example -->
```svelte
<script>
import { WhenVisible } from '@inertiajs/svelte'

export let stats
</script>

<div>
    <h1>Dashboard</h1>

    <!-- stats prop is loaded only when this section scrolls into view -->
    <WhenVisible data="stats" buffer={200}>
        <div>
            <p>Total Users: {stats.total_users}</p>
            <p>Revenue: {stats.revenue}</p>
        </div>

        <svelte:fragment slot="fallback">
            <div class="animate-pulse">Loading stats...</div>
        </svelte:fragment>
    </WhenVisible>
</div>
```

## Server-Side Patterns

Server-side patterns (Inertia::render, props, middleware) are covered in inertia-laravel guidelines.

## Common Pitfalls

- Using traditional `<a>` links instead of Inertia's `<Link>` component (breaks SPA behavior)
- Forgetting to add loading states (skeleton screens) when using deferred props
- Not handling the `undefined` state of deferred props before data loads
- Using `<form>` without preventing default submission (use `<Form>` component or `on:submit|preventDefault`)
- Forgetting to check if `<Form>` component is available in your Inertia version