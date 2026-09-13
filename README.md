# turbo-localappdata-repro

Minimal reproduction: Turborepo's builtin pass-through list (`BUILTIN_PASS_THROUGH_ENV` in `crates/turborepo-env/src/lib.rs`) contains the Windows shell path variables — `APPDATA`, `PROGRAMDATA`, `SYSTEMROOT`, `SYSTEMDRIVE`, `USERPROFILE`, `HOMEDRIVE`, `HOMEPATH`, `WINDIR`, `ProgramFiles` — but **not `LOCALAPPDATA`**. Under the default `strict` env mode, tasks never receive it.

Verified on Windows 11 with turbo canary:

```text
$ npx turbo@canary --version
2.10.13-canary.5
```

The shell that starts turbo has the variable set:

```text
> echo %LOCALAPPDATA%
C:\Users\<you>\AppData\Local
```

## Reproduce

```sh
npx turbo@canary run probe --ui=stream
```

```text
probe:probe: cache bypass, force executing 865fddf7dbb358ca
probe:probe:
probe:probe: > probe@1.0.0 probe <repo>\packages\probe
probe:probe: > node check.mjs
probe:probe:
probe:probe: LOCALAPPDATA=MISSING
```

`packages/probe/check.mjs` prints the value the task process actually received.

## Workaround

Add `globalPassThroughEnv` to `turbo.json`:

```json
{
  "$schema": "https://turborepo.dev/schema.json",
  "globalPassThroughEnv": ["LOCALAPPDATA"],
  "tasks": {
    "probe": { "cache": false }
  }
}
```

```text
$ npx turbo@canary run probe --ui=stream
probe:probe: LOCALAPPDATA=C:\Users\<you>\AppData\Local
```

## Impact

`pnpm.exe` stalls without this variable: it resolves its package-manager env directory under `%LOCALAPPDATA%\pnpm\...`, so `turbo run dev` reports the task as executing but never spawns the dev server (`vite`, `uvicorn`, …). Tasks in the same workspace using npm as the package manager are unaffected, which is why this surfaces only in pnpm workspaces.

When the variable is present but points to a non-existent path, pnpm fails fast instead of hanging — so the missing-variable case is detectable:

```text
Error:   × create the package-manager env directory at Z:\fake\pnpm\global\v11
  ╰─▶ The system cannot find the path specified. (os error 3)
```

Pass-through variables do not feed the task hash, so adding `LOCALAPPDATA` to the builtin list cannot destabilize cache keys: `turbo run build --dry=json` produces an identical task hash with two different `LOCALAPPDATA` values.
