# Auto-rollback deployer

This is a work-in-progress deployment tool I'm developing for [https://niteo.co/](Niteo). The main distinguishing feature as of now is that it automatically does a rollback if the new system failed in some ways. Notably it protects against:

- Messing up the network config
- Removing your SSH key from the authorized keys
- The activation script failing in any way
- The boot activation failing in any way
- The system crashing during the deployment

## How to use it

Note: This is just to demonstrate, this will certainly change in the future

Write a file like `example/default.nix`, then build the deployment script and call it
```
$ nix-build example/default.nix
these derivations will be built:
  /nix/store/pm0dl01vzhjiy2ghnz7c7bzgq09l9zl5-etc-hosts.drv
  /nix/store/syvikkxpvxsdij0p0cjjkwy19rw2iqny-unit-nscd.service.drv
  /nix/store/c1yqnbngp5da902vhsv8rg419id9wil5-system-units.drv
  /nix/store/rbgshjafl0lb7y49v13q4r6gbwsxx9pf-etc-hostname.drv
  /nix/store/qhg4f8mz5jbfydh3x3hazcag0q6a5lmz-etc.drv
  /nix/store/kry2am4bc8zmah4mn1s9azbc5j77mahh-nixos-system-example-20.03pre-git.drv
  /nix/store/9svlsc4sw6kw7db5b024qvnhnrl9jyzf-deploy-foo.example.com.drv
  /nix/store/bii36sla8vwq9k52n6w5f1gfx9pfxyvi-deploy.drv
building '/nix/store/rbgshjafl0lb7y49v13q4r6gbwsxx9pf-etc-hostname.drv'...
building '/nix/store/pm0dl01vzhjiy2ghnz7c7bzgq09l9zl5-etc-hosts.drv'...
building '/nix/store/syvikkxpvxsdij0p0cjjkwy19rw2iqny-unit-nscd.service.drv'...
building '/nix/store/c1yqnbngp5da902vhsv8rg419id9wil5-system-units.drv'...
building '/nix/store/qhg4f8mz5jbfydh3x3hazcag0q6a5lmz-etc.drv'...
building '/nix/store/kry2am4bc8zmah4mn1s9azbc5j77mahh-nixos-system-example-20.03pre-git.drv'...
building '/nix/store/9svlsc4sw6kw7db5b024qvnhnrl9jyzf-deploy-foo.example.com.drv'...
building '/nix/store/bii36sla8vwq9k52n6w5f1gfx9pfxyvi-deploy.drv'...
/nix/store/zs6zz7ky2a5lxdv6jcg13chgsvk24338-deploy
$ ./result
[foo.example.com] Copying closure to host..
[foo.example.com] copying 6 paths...
[foo.example.com] copying path '/nix/store/5hmr5hla64w09v4fgsllkb66snkffj22-etc-hosts' to 'ssh://root@138.68.83.114'...
[foo.example.com] copying path '/nix/store/9434s915psr7c3rw4rb5ybcpai5n74gm-etc-hostname' to 'ssh://root@138.68.83.114'...
[foo.example.com] copying path '/nix/store/xsznipc9bmi2y7bcmv3yv6vm8sw42r8r-unit-nscd.service' to 'ssh://root@138.68.83.114'...
[foo.example.com] copying path '/nix/store/z7bz5jqi1jpvm0p5iwfaxw9r48mkm3p1-system-units' to 'ssh://root@138.68.83.114'...
[foo.example.com] copying path '/nix/store/0lni7y6hman5xyaa4a4f2f4sbvci8g22-etc' to 'ssh://root@138.68.83.114'...
[foo.example.com] copying path '/nix/store/0rw6f5qfjin9112dlcwk8268z1k1vf43-nixos-system-example-20.03pre-git' to 'ssh://root@138.68.83.114'...
[foo.example.com] Triggering system switcher..
[foo.example.com] Trying to confirm success..
[foo.example.com] Successfully activated new system!
```

Here is an example of a messed up network config:
```
[foo.example.com] Copying closure to host..
[foo.example.com] copying 5 paths...
[foo.example.com] copying path '/nix/store/0imhl89rfy91k1ks4arf6jmfiy4rwqsz-switch' to 'ssh://root@138.68.83.114'...
[foo.example.com] copying path '/nix/store/jakl1qb54dwkw28wpv8jxyvv94bpcq2c-unit-network-setup.service' to 'ssh://root@138.68.83.114'...
[foo.example.com] copying path '/nix/store/yr26fj6hsfvdhc798mrc92bmgfx1x0n8-system-units' to 'ssh://root@138.68.83.114'...
[foo.example.com] copying path '/nix/store/517yb981jsllyfa87apd3yq0dzkn333z-etc' to 'ssh://root@138.68.83.114'...
[foo.example.com] copying path '/nix/store/2gsp2fgy9ihi6wn577gyi0k04ydzvzi7-nixos-system-example-20.03pre-git' to 'ssh://root@138.68.83.114'...
[foo.example.com] Triggering system switcher..
[foo.example.com] Trying to confirm success..
[foo.example.com] Failed to activate new system! Rolled back to previous one
```

## How it works

The basic idea is to first do a `nixos-rebuild test` on the target machine, after which we try to connect to the machine again to confirm that it worked. If we can't make this confirmation, a rollback is issued.
