---
title: Permission Configuration
description: Configure command permissions, groups, player overrides, and permission metadata in SteelMC.
sidebar:
  order: 4
---

SteelMC permissions are configured through `config/permission/groups.toml` and can also be edited at runtime with `/perms`.

Permissions control command access, command suggestions, and the command tree sent to each client. The same system also supports contextual rules and typed metadata values for server features and plugins.

## Files

Steel uses three permission files:

| File | Purpose |
| ---- | ------- |
| `config/permission/groups.toml` | Server-wide permission groups and default groups |
| `config/permission/collections.toml` | Optional collection overrides edited by `/perms collection` |
| `<save_root>/global/player_permissions.toml` | Per-player assigned groups, direct overrides, and metadata |

Edit `groups.toml` for normal server policy. A collection declared in `collections.toml` replaces the generated or code-declared collection with the same id; its entries are never merged with them and it is not updated automatically. Editing or exporting a collection that was not already configured prints a warning about this precedence. The player permissions file is under the configured world `save_path`, which defaults to `saves`. It is managed by Steel and `/perms`; manual edits should only be needed for recovery or bulk migration while the server is offline.

## Default Groups

A new server uses the vanilla permission levels as groups: `all`, `moderators`, `gamemasters`, `admins`, and `owners`. Every player receives `all` because it is listed in `default_groups`. The required `owners` group is assigned by `/op`. `op_group` names that single group, and `command_block_groups` lists the groups granted to command blocks. Both can be changed at runtime with `/perms groups op set` and `/perms groups command_block add|remove`.

Each level inherits the previous one, so `owners` also receives everything from `admins`, `gamemasters`, `moderators`, and `all`. The levels come from extracted vanilla data; each one references a code-owned collection such as `#minecraft:gamemasters` rather than listing commands directly.

`all` has no access to `/perms` and receives no management authority. Every other level can administer its own permissions and group, and `admins` (therefore `owners` too) can also manage the `all` group. See [Management authorities](#management-authorities).

Add server roles as tables named `[groups.<name>]`. Each group can contain permission rules, metadata, and parent groups to inherit from. A [complete example](#complete-example) later on this page shows how these pieces fit together.

:::caution
Do not put a higher level such as `owners` in `default_groups` unless every player should receive its permissions.
:::

## Permission Keys

Permission keys are written as `namespace:path`, where the path is made of dot-separated lowercase segments. A namespace-less key such as `command.list` is also valid and matches every namespace:

```text
minecraft:command.give
steel:command.fly
plugin:region.build
```

:::caution
Permission key syntax is strict. Segments can contain only lowercase ASCII letters, digits, `_`, and `-`. Empty segments and uppercase letters are invalid.
:::

Wildcards are allowed only as the final segment:

| Key | Meaning |
| --- | ------- |
| `*` | Matches every permission |
| `minecraft:command.*` | Matches descendants such as `minecraft:command.give` |
| `minecraft:command.give` | Matches only that exact key |

`minecraft:command.*` matches `minecraft:command.give`, but it does not match `minecraft:command` itself.

:::note
A wildcard matches descendants only. Grant both `minecraft:command` and `minecraft:command.*` if a system defines meaningful permissions at both levels.
:::

## Command Permissions

Most built-in commands automatically use this permission shape:

```text
<namespace>:command.<command>
```

Examples:

| Command | Permission |
| ------- | ---------- |
| `/give` | `minecraft:command.give` |
| `/fly` | `steel:command.fly` |
| `/tp` and `/teleport` | `minecraft:command.teleport` |
| `/xp` and `/experience` | `minecraft:command.experience` |
| `/perms` | `steel:command.perms` |

Commands are granted through permission collections rather than a special default-access flag. For example, `minecraft:command.list` is a member of `#minecraft:all`, which the default `all` group allows. An explicit deny can still override that grant.

Some commands expose narrower subcommand or value permissions. For example, `/tick freeze` can be granted with either `minecraft:command.tick` or `minecraft:command.tick.freeze`.

`/gamemode` has value-specific permissions:

| Action | Permission |
| ------ | ---------- |
| Any gamemode change | `minecraft:command.gamemode` |
| Survival only | `minecraft:command.gamemode.survival` |
| Creative only | `minecraft:command.gamemode.creative` |
| Adventure only | `minecraft:command.gamemode.adventure` |
| Spectator only | `minecraft:command.gamemode.spectator` |

A broader permission grants the child action, but a more specific deny can override it. For example, allow `minecraft:command.gamemode` and deny `minecraft:command.gamemode.creative` to allow all gamemode changes except creative.

### Management authorities

Using `/perms` requires two separate things:

1. Access to the command itself, granted by `steel:command.perms`. This permission is declared in `minecraft:moderators`, so every vanilla level except `all` can reach `/perms`.
2. An authority for the action being performed. These authorities live in the vanilla collections, not in the groups:

   | Authority | Needed for |
   | --------- | ---------- |
   | `steel:permission.manage.<namespace>.<path>` | `user allow`/`deny`/`unset`, `group allow`/`deny`/`unset`, and collection edits |
   | `steel:permission.manage.minecraft.<level>` | Adding or removing a `#minecraft:<level>` collection rule |
   | `steel:permission.group.<name>` | Group assignment, creation, deletion, priority, inheritance, default groups, `op_group`, and `command_block_groups` |
   | `steel:permission.metadata` | Reading and editing metadata |
   | `steel:permission.manage.*` | `/perms reload` |

Each vanilla level above `all` carries the authorities for its own permissions, its collection, its group, and metadata. `minecraft:admins` also carries `steel:permission.group.all`, so `admins` and `owners` can manage the default group. Levels inherit downward, so `admins` can manage `admins`, `gamemasters`, `moderators`, `all`, and their permissions, but not `owners`. `owners` also receives `steel:permission.manage.*` and `steel:permission.group.*` in its `minecraft:owners` collection, so it can administer everything including future plugin keys. `all` receives no management authority.

Permissions declared by code also grant their matching `steel:permission.manage.<namespace>.<path>` authority in the same collection. This covers built-in Steel commands and permissions contributed with `CommandRegistry::declare_permission`, including plugin contributions, so they can be edited with `/perms` without adding authorities by hand.

A management authority authorizes itself: editing `steel:permission.manage.minecraft.command.give` requires that same key, not a nested `steel:permission.manage.steel.permission.manage...` key. This also applies to generated authorities such as `steel:permission.manage.steel.permission.group.admins`.

Having a management authority without `steel:command.perms` is not enough: the command requirement is checked before the action handler runs.

A group created in `groups.toml` receives no management authority by default. To let it administer a level, add that level's collection to its `allow` list, for example `#minecraft:gamemasters`.

## Group Configuration

Groups are tables under `[groups.<name>]`.

```toml
[groups.moderator]
priority = 10
inherits = []
allow = [
  "minecraft:command.teleport",
  "minecraft:command.gamemode.spectator",
]
deny = [
  "minecraft:command.stop",
]
metadata = []
```

Group names must be valid permission segments: lowercase letters, digits, `_`, and `-`.

### Group Inheritance

Groups can inherit the permissions and metadata of other groups:

```toml
[groups.helper]
priority = 5
inherits = []
allow = ["minecraft:command.teleport"]
deny = []
metadata = []

[groups.moderator]
priority = 10
inherits = ["helper"]
allow = ["minecraft:command.gamemode.spectator"]
deny = []
metadata = []
```

Members of `moderator` receive both permissions. The inherited teleport rule keeps the `helper` priority of `5`; it does not take on the `moderator` priority of `10`.

Inheritance is transitive, and each inherited group contributes at most once. Cycles and references to missing groups make the configuration invalid.

### Contextual Rules

Permission and metadata rules can include context selectors:

```toml
[groups.builder]
priority = 5
inherits = []
allow = [
  "plugin:region.build{world=lobby:spawn,plugin:region=market}",
  "minecraft:command.gamemode{domain=lobby}",
]
deny = [
  "minecraft:command.gamemode.creative{world=lobby:spawn}",
]
metadata = []
```

Built-in context keys:

| Context | Example | Meaning |
| ------- | ------- | ------- |
| `domain` | `domain=lobby` | Applies only inside one [domain](../../reference/terminology#domain) |
| `world` | `world=lobby:spawn` | Applies only inside one loaded [world](../../reference/terminology#world) |

Custom contexts can be provided by plugins or server subsystems. Unqualified keys such as `region=spawn` are valid, but namespaced keys such as `plugin:region=spawn` are preferred for plugin-owned contexts.

Multiple selectors are combined with AND:

```text
plugin:region.build{world=lobby:spawn,plugin:region=market}
```

:::caution
Context syntax is strict. Each key can appear only once in an expression.

`world=lobby:spawn` already includes the `lobby` domain and therefore matches rules scoped to `domain=lobby`. Writing both is unnecessary. If both are present, they must agree; `{domain=survival,world=lobby:spawn}` is invalid.

Values cannot be empty and cannot contain whitespace, `{`, `}`, `,`, or `=`. Domain names use Minecraft identifier-namespace syntax, and worlds must be written as `<domain>:<world>`.
:::

Context specificity is additive. Global rules are least specific, a domain or custom selector adds one level, and a world adds two because it identifies both a domain and a loaded world. A rule with several matching selectors is therefore more specific than a rule with only one of them.

## Metadata

Metadata is typed data resolved by the same group and context model as permissions. Values can be booleans, integers, or strings.

```toml
[groups.vip]
priority = 20
inherits = []
allow = []
deny = []
metadata = [
  { key = "plugin:homes", value = 10 },
  { key = "plugin:homes{domain=lobby}", value = 3 },
  { key = "plugin:chat/color", value = "gold" },
  { key = "plugin:can_fly", value = true },
]
```

Metadata keys must be namespaced identifiers such as `plugin:homes` or `plugin:chat/color`. The system that owns a key decides what its value means: for example, a homes plugin could interpret these rules as a limit of 10 homes normally and 3 inside the `lobby` domain. Steel stores and resolves the values but does not implement homes, chat colors, or flight merely because these example keys exist.

Metadata resolution considers only entries with the requested exact metadata key. Among matching entries, Steel chooses:

1. The most specific context.
2. A direct player value over a group value when specificity ties.
3. The higher group priority when group values tie.
4. The last entry in the effective metadata set when everything else ties.

Unlike permissions, metadata has no allow/deny state and metadata keys do not use wildcard matching. Setting a value replaces only the entry with the same metadata key and exact context; unsetting it likewise removes only that exact entry. A global value and a contextual value for the same key can coexist.

## Conflict Resolution

Unset permissions are denied. When multiple rules match, Steel chooses the winning rule in this order:

1. More specific permission key wins.
2. More specific context wins.
3. Direct player overrides beat group rules when specificity ties.
4. Higher group priority wins when group rules tie.
5. Deny wins when everything else ties.

This means a specific deny can override a broad allow:

```toml
[groups.staff]
priority = 10
inherits = []
allow = ["minecraft:command.*"]
deny = ["minecraft:command.stop"]
metadata = []
```

The player can use most Minecraft commands, but not `/stop`.

## Complete Example

This configuration adds two custom groups that borrow vanilla collections:

```toml
default_groups = ["all"]

[groups.build]
priority = 5
inherits = []
allow = ["#minecraft:gamemasters"]
deny = []
metadata = [
  { key = "example_homes:max", value = 5 },
]

[groups.staff]
priority = 10
inherits = []
allow = ["#minecraft:moderators"]
deny = []
metadata = []
```

- Everyone receives `all`.
- Members of `build` can use the gamemasters commands and manage that level and below through `/perms`.
- Members of `staff` can use and manage the moderators level.
- `/op` assigns the required `owners` group, which inherits every vanilla level and can manage all of them.

The `example_homes:max` entry is illustrative. It has an effect only when a plugin that owns that key reads it. Replace it with metadata supported by the plugins you actually install.

## Runtime Commands

Use `/perms` to inspect and edit permissions while the server is running. Its general shape is:

```text
/perms <user|group|groups|collection> ...
/perms reload
```

Start by inspecting the current configuration:

```text
/perms user <targets> info
/perms group <group> info
/perms groups list
```

These commands show direct player settings, group rules and inheritance, the available groups with their `default`/`op`/`command_block` roles, and the effective contents of collections. After editing the files by hand, use `/perms reload` to reread them and refresh online players; a failed reload leaves the previous configuration active. See [Permission commands](../../commands/permissions) for every operation, required permission, and example.

## Operators

`/op <targets>` and `/deop <targets>` are backed by the permission system:

- `/op` adds the group named by `op_group` (`owners` by default).
- `/deop` removes that group.
- `owners` inherits every vanilla level and can manage all of them through `/perms`.
- The `op_group` group is required and cannot be deleted with `/perms`. Change it first with `/perms groups op set <group>` if you need a different one.

Targets can be online players, known offline players, or profile names the server can resolve.

## Client Command Tree

Players only receive command nodes they are allowed to use. This affects:

- visible commands in the client command tree
- tab completion
- server-side command suggestions
- the client gamemode switcher

Console and RCON bypass permission checks.
