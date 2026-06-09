# NBenchmark Skills

AI agent skills for [NBenchmark](https://github.com/nbenchmark/nbenchmark), a lightweight async-native .NET benchmarking library.

## What's Included

| Skill | Description |
|---|---|
| **nbenchmark** | Core benchmarking with Quick mode (`Benchmark.Run` / `RunAsync`) and Suite mode (`BenchmarkSuite` fluent builder) |
| **nbenchmark-host** | Host mode for dedicated benchmark projects - attribute-based discovery, CLI flags, dependency injection, and CI/CD regression detection |
| **nbenchmark-reporters** | Reporter pipeline for console, JSON, Markdown, and CSV output, plus custom reporter creation |
| **nbenchmark-troubleshooting** | Diagnosing analyzer warnings, incorrect results, discovery issues, and tuning measurement parameters |

## Installation

The easiest way to add these skills to your project is via [skills.sh](https://skills.sh):

```sh
npx skills@latest add nbenchmark/skills
```

This launches an interactive picker where you select the skills you want and which coding agents to install them on.

### Manual Installation

If you prefer to set things up manually, copy the skill directories you need into your project's agent config:

**For Claude Code** - copy into `.claude/skills/`:

```sh
# From your project root
mkdir -p .claude/skills
cp -r skills/nbenchmark .claude/skills/
cp -r skills/nbenchmark-host .claude/skills/
cp -r skills/nbenchmark-reporters .claude/skills/
cp -r skills/nbenchmark-troubleshooting .claude/skills/
```

**For opencode** - copy into `.opencode/skills/`:

```sh
# From your project root
mkdir -p .opencode/skills
cp -r skills/nbenchmark .opencode/skills/
cp -r skills/nbenchmark-host .opencode/skills/
cp -r skills/nbenchmark-reporters .opencode/skills/
cp -r skills/nbenchmark-troubleshooting .opencode/skills/
```

**For other agents** that follow the [skills.sh convention](https://github.com/mattpocock/skills), each skill is a `SKILL.md` file inside a named directory under `skills/`. Adapt the directory structure to your agent's convention.

## License

These skills are provided as-is for use with NBenchmark projects.
