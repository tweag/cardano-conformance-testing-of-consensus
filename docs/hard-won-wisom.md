# Hard Won Wisdom

Useful notes and pointers that we would pass to out past selves for developing
in the Cardano ecosystem.

## Places

- The Cardano package repository is called _CHaP_ (cardano-haskell-packages): <https://chap.intersectmbo.org/>.
- Developer documentation: <https://developers.cardano.org/>
- Ouroboros consensus documention: <https://ouroboros-consensus.cardano.intersectmbo.org/>
- Ouroboros network haddocks: <https://ouroboros-network.cardano.intersectmbo.org/>

## Development Environment

The (apparently) recommended way of developing Cardano is using Nix. This comes
with it own set of quirks depending on what you want to do.

**Preventing GC of your nix shell:** make sure to set `keep-derivations = true`
and `keep-outputs = true` on your Nix configuration file. Next, gc-roots need to
be created for the nix-shell. Using `nix-direnv` takes care of this automatically,
but you could also manually create a gc-root using something like:
`nix-instantiate shell.nix --indirect --add-root $DIR/.nix-gc-roots/shell.drv`.
References:

- <https://ianthehenry.com/posts/how-to-learn-nix/saving-your-shell/>
- <https://discourse.nixos.org/t/why-does-nix-direnv-recommend-setting-nix-settings-keep-outputs/31081/2>

When using `nix-direnv` to hook your editor to the nix shell is important to
note that the derivation is probably hooked to `.cabal` and `cabal.project`.
If one updates this files and finds problems, the first debugging step should
be running `direnv reload` for the cached nix shell to update in case it did not.

### Doom Emacs

Enable the `direnv`, `lsp` and `(haskell +lsp)` modules in `init.el`.

When entering a new project, run the following in the repository root to enable
`direnv` to use the nix shell:

```
echo "use flake" > .envrc && direnv allow .
```

Here are some useful configuration options for `lsp-haskell` that can be set in
`config.el`:

```elisp
(after! lsp-haskell
  ;; Use the nix shell provided executable (default: haskell-language-server-wrapper).
  ;; as expected to work with GHCup executables.
  (setq lsp-haskell-server-path "haskell-language-server") 
  ;; Set the formatter (default: ormolu); `ouroboros-consensus` uses fourmolu.
  (setq lsp-haskell-formatting-provider "fourmolu") 
  ;; Use the nix shell provided executable to ensure CI compatibility.
  (setq lsp-haskell-plugin-fourmolu-config-external t) 
  ;; Convenient to reference across components,
  ;; but increases memory footprint substantially.
  (setq lsp-haskell-session-loading "multipleComponents") 
  )
```

## Merging code

When developing across many Cardano packages the (currently) recommended
workflow is to pick a specific `cardano-node` release tag (having a comprehensive
change log) and develop against the appointed dependencies.

For this to work, you can link specific dependency commit tags by using a
`source-repository-package` stanza in `cabal.project`.

## Profiling

Some nix shells provide a "profiling" derivation you can access using
`nix develop .#profiled`.

## Generating Documentation

`cabal-hoogle` and `hoogle` can be invoked in some of the Cardano repositories
to build a local hoogle server.
