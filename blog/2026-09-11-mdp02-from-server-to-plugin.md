---
layout: post
title: "From Server to Plugin in One Session"
date: 2026-09-11
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [intellij, lsp, plugin, esbuild, kotlin, lsp4ij]
---

# From Server to Plugin in One Session

The pages-lsp server has been sitting in `packages/pages-lsp/` since last week — completion, diagnostics, hover, rename, cross-file references, jq expression intelligence — all working. Today's job was getting it into IntelliJ.

## The Bundling Problem

An LSP server that lives as unbundled ESM in a `node_modules` tree can't ship inside a plugin. The fix is esbuild: one CJS bundle, one file, 1.2MB. The build script is ten lines. The interesting part was proving it worked — spawning the bundle as a subprocess, sending LSP JSON-RPC over stdio, and verifying the handshake, completion, diagnostics, and hover all function from the single file. Those four integration tests are the real deliverable; the esbuild config is just plumbing.

## The Community Edition Surprise

I started with IntelliJ's built-in LSP API — `com.intellij.platform.lsp.api.LspServerSupportProvider`. The classes are documented, the API is clean, and the code compiled against IntelliJ Platform 2024.1. Then it didn't. The `intellij.platform.lsp` module doesn't exist in Community Edition. Searched every JAR in the IC distribution: nothing. Only `com.intellij.internal.sandbox.LSPServer` — an internal class, not the public API.

The documentation doesn't say this. It describes the API as part of the IntelliJ Platform, which is the common layer. But the module is only bundled in Ultimate. The pivot was to LSP4IJ — Red Hat's LSP client that works everywhere. Simpler API, too: `LanguageServerFactory` creates a `StreamConnectionProvider`, and `fileNamePatternMapping` in plugin.xml handles activation. No custom file type registration needed.

## Where the Plugin Lives

The original design spec placed IDE plugins in blocks-ui. The issue spec proposed a new `casehubio/casehub-intellij` repo. I challenged both — the pages repo already has Java in `backend/`, Gradle is self-contained in a subdirectory, and the server bundle the plugin ships is a build artifact two directories up. The answer is `plugins/intellij/`. One repo, one commit when the server API changes.

## The Rename Trap

The Plugin Verifier rejects plugin IDs containing the word "intellij" — a marketplace rule to prevent brand confusion. Fair enough. I renamed `io.casehub.intellij` to `io.casehub.yaml`. Claude caught what I missed during code review: the `replace_all` that changed the plugin ID also changed the `factoryClass` attribute in plugin.xml from `io.casehub.intellij.lsp.CaseHubLanguageServerFactory` to `io.casehub.yaml.lsp.CaseHubLanguageServerFactory`. The Kotlin source files still used the original package. The plugin would have loaded, found nothing at that class path, and silently failed. A one-character difference between "builds successfully" and "doesn't work."

## The JDK 26 Error That Isn't an Error

Gradle printed `26.0.2` as the entire error message. No exception type, no file reference, no "Caused by". Just a version number. The Kotlin compiler embedded in Gradle 8.x can't parse JDK 26 version strings — `JavaVersion.parse()` throws `IllegalArgumentException` with the version as the message. Gradle catches it and prints... the message. You need `--stacktrace` to see what's actually happening. The fix is running on JDK 21, but reaching that fix requires knowing the problem is JDK version parsing and not, say, a misconfigured Gradle plugin.

## What's Shipping

A 192KB plugin ZIP, verified compatible across IntelliJ 2024.2 through 2025.2. Open a `.page.yaml` file, get completion narrowed by discriminator value, schema diagnostics, hover, rename across files. The server is 1.2MB of bundled TypeScript; the plugin is 56KB of Kotlin. The ratio tells the story — the plugin is a launcher, the intelligence is in pages-lsp.

Next: domain schemas. The server currently knows Page format only. Case definitions, SWF workflows, HTN plans, and org structures each need Zod schemas generated from their TypeScript interfaces and registered with the schema registry. That's blocks-ui work — a different slot, a different session.
