
## GHCJS Commentary

- GHCJS is a Haskell to JavaScript compiler written in Haskell, based on the GHC compiler
- Principal author is Luite Stegeman
- It can not compile itself, you need GHC to compile GHCJS

- Dependencies:
  - GHC (see above) and Stack/Cabal to manage building GHCJS
  - To boot
    - ?
  - To run, GHCJS depends on these executables being available on your system:
    - Node JS (version?)
    - Other?
  - Current GHCJS master can be compiled by GHC 7.10.2

- Libraries/executables
  - The ghcjs cabal package exposes a library (also called 'ghcjs' as per Cabal convention), and the following executables
    - Main stuff:
      - ghcjs (the compiler!)
      - ghcjs-pkg (manage package databases which can be passed to ghcjs)
      - ghcjs-boot (need to be called before ghcjs is usable)
    - Nice to have:
      - haddock-ghcjs (doc generator)
      - hsc2hs-ghcjs (?)
      - ghcjs-run (purpose?)
  - It also exposes a test suite
  - It depends on the 'ghc' package (which always has the same version as the compiler that compiled GHCJS)
  - The ghcjs package version is lower than the ghc (library/compiler dep) version

- Source files:
    https://github.com/ghcjs/ghcjs.git (the compiler)
      - src-bin
        Binaries (see above)
          - Main (implements `ghcjs`, calls into `Compiler.Program`, see below)
          - Boot (implements `ghcjs-boot`)
            - Creates boot settings from commmand line
            - Calls initBootEnv to obtaina boot env, which is configuration plus knowledge of GHC/GHCJS paths and binaries
              - Binaries called into/used
                ghcjs
                ghcjs-pkg
                ghcjs-run
                ghc
                ghc-pkg
                Cabal
                node
            - Performs a series of actions, notably
              installBuildTools
              bool (e ^. beSettings . bsDev) installDevelopmentTree installReleaseTree
              initPackageDB
              cleanCache
              installRts
              installEtc
              installDocs
              installTests
              copyGhcjsPrim
              copyIncludes
              let base = e ^. beLocations . blGhcjsLibDir
              setenv "CFLAGS" $ "-I" <> toTextI (base </> "include")
              installFakes
              installStage1
              unless (e ^. beSettings . bsStage1Unbooted) $ do
                removeFakes
                unless (e ^. beSettings . bsQuick) installStage2
              when (e ^. beSettings . bsHaddock) buildDocIndex
              liftIO . printBootEnvSummary True =<< ask
              unless (e ^. beSettings . bsStage1Unbooted) addCompleted

      - src
        Most code is here

        - Top-level:
          - *Compiler.GhcjsProgram*
              - GHCJS-specific parts of front end (see Compiler.Program)
          - *Compiler.Program*
              - GHCJS frontend (CLI interface)
          - *InteractiveUI*
              - Implements interactive mode `ghcjs --interactive` and `ghci`
              - Main functions `Compiler.InteractiveUI( interactiveUI, ghciWelcomeMsg, defaultGhciSettings )`
              - Uses modules like
                - *InteractiveEval*
                - *GhciMonad*
                - *GhciTags*
          - *DriverPipeline*
              - Behind much of `ghcjs --make` (multi-file compilation), which in turn is behind `cabal/stack build`
              - Forked from ghc package (*both* Compiler.DriverPipeline and the original "ghc".DriverPipeline are used)
              - Main function: `compileFile`

        - Codegen
          Main entry-point (TODO verify!):
            Gen2.Generator (generate), called from Compiler.Variants and Compiler.InteractiveEval
          - Important modules:
              Gen2.Base
              Gen2.Rts
              Gen2.Object
              Gen2.Linker
              Gen2.DynamicLinking

              Gen2.Optimizer/Compactor
                Optimization etc. Mainly pure modules
                Gen2.Dataflow
                  User by Gen2.Optimizer
          - Several Gen2 modules doing IO:
                archive
                cache
                compactor
                dynamicLinking
                foreign
                linker
                object
                prim
                rts
                shim
                TH
                Utils
        - Low-level/Utility:
          - *Compiler.GhcjsHooks*
            - Looks like things that would be used with the GHC Hooks API, but I see no evidence of this
            https://ghc.haskell.org/trac/ghc/wiki/Ghc/Hooks

          - *Compiler.GhcjsPlatform* utility to set a fake "platform" as required by GHC API
          - *Compiler.Info*
            - Provide compiler version and dirs (in IO)
          - *Compiler.Settings*
            - Settings passed from env/config/cli interface, pure type: *GhcjsSettings*
            - Impure type, representing an entity that can execute TH (i.e. nodejs): *ThRunner*
            - Impure type, misc state gathered during compilation: *GhcjsEnv*
          - *Compiler.Utils*
              Utility, IO etc
          - *Compiler.Variants*
              ?
          - *Compiler.JMacro*
              - JavaScript AST/parser/prettyprinter
                - Copied from the 'jmacro' package
                - Main types: JSStat/JSExpr
                - QuasiQuoters to inline syntactically correct JS in Haskell files:
                  - Compiler.JMacro.QQ(jmacro, jmacroE), aka Compiler.JMacro(j, je), generates JSStat/JSExpr respectively

    https://github.com/ghcjs/shims (runtime system and JS code)
    https://github.com/ghcjs/ghcjs-prim (OUTDATED)
    https://github.com/ghcjs/ghcjs-base
      A JS side library with low-level HS-JS interaction, wrappers for common JS APIs etc
      Nothing magical about this library (i.e. not wired in magically in the compiler)

- What does `ghcjs` do
  - TODO
  - Use GHC API to set various DynFlags
    https://downloads.haskell.org/~ghc/7.6.1/docs/html/libraries/ghc-7.6.1/DynFlags.html#t:GhcLink
    Lots of these refer to native compilation, so I assume these behaviors are overridden
    GhcMode
      https://downloads.haskell.org/~ghc/7.6.1/docs/html/libraries/ghc-7.6.1/DynFlags.html#t:GhcMode
      Single vs multi-file compilation (are both supported by GHCJS?)
      Usually CompManager or MkDepend, byt also OneShot
    HscTarget
      https://downloads.haskell.org/~ghc/7.6.1/docs/html/libraries/ghc-7.6.1/DynFlags.html#t:HscTarget
      HscInterpreted for eval/interactive mode, otherwise the default
    GhcLink
      https://downloads.haskell.org/~ghc/7.6.1/docs/html/libraries/ghc-7.6.1/DynFlags.html#t:GhcLink
      Set to LinkInMemory (interactive/eval) or LinkBinary (compilation)

  - Eventually, one of the following will be called
    doMake for `ghc --make`
    doShowIface
    etc (for full list, see the case statement dispatching on PostLoadMode, or the PostLoadMode definition)

- Booting
  Performs lots of different setup steps (TODO overview)
  No hardcoded paths, they are all set in the BootEnv type (TODO check more closely which paths are relevant to GHCJS)

- Testing?

- Docs/Low-level docs?

- Use with Stack
  TODO investigate exactly what happens here
  Declared as
    compiler: ghcjs-0.2.0.20151230.3_ghc-7.10.2
    compiler-check: match-exact
    setup-info:
     ghcjs:
      source:
       ghcjs-0.2.0.20151230.3_ghc-7.10.2:
        url: "https://github.com/nrolland/ghcjs/releases/download/v.0.2.0.20151230.3/ghcjs-0.2.0.20151230.3.tar.gz"

- TODO what is the purpose of the various data-files entries in the cabal file?
