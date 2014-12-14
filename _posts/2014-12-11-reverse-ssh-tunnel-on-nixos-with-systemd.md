---
layout: post
title: Reverse SSH Tunnel on NixOS
---

Like many SSH users, I often find myself stuck behind some firewall or NAT that can't be reasoned with. Starting a reverse SSH tunnel with the -R flag is not difficult, but it's tedious to do more than once, and easy to forget to do beforehand. Plus, I'm pretty lazy. Wouldn't it be nice if we could set that up automatically?

## One Google Search Later

Fortunately, I'm not the only lazy person who uses Linux; some clever people are already leveraging systemd to do the legwork work for them. I came across an <a href="http://blog.kylemanna.com/linux/2014/02/20/ssh-reverse-tunnel-on-linux-with-systemd/">excellent post by Kyle Manna</a> that explains how to acomplish reverse SSH forwarding on Arch Linux with systemd. And since I'm lazy, rather than trying to figure out everything myself I just took his systemd unit file and ran with it.

## From systemd Unit File to NixOS Module

If you just want to try out reverse SSH tunneling on NixOS yourself, the code for the NixOS module is <a href="https://github.com/auntieNeo/nixrc/blob/master/services/ssh-phone-home.nix">here in my nixrc repository</a>. For those of you interested in writing a simple systemd service for NixOS, read on.

### The Wrong Way

My first instinct when starting to write the NixOS module was to dig around in nixpkgs for a similar NixOS service module. A quick peek under `nixpkgs/nixos/modules/services/networking` and I found a relatively simple module for the GNU implementation of an SSH 2 daemon. Here is the relevant code from that module, heavily redacted from <a href="https://github.com/NixOS/nixpkgs/blob/4668f374448dce5ed92707eb2d20d77bf7380b47/nixos/modules/services/networking/ssh/lshd.nix">the original</a> for brevity:

{% highlight nix %}
{ config, lib, pkgs, ... }:
with lib;
let
  inherit (pkgs) lsh;
  cfg = config.services.lshd;
in
{
  options = {
    # options REDACTED
  };
  config = mkIf cfg.enable {
    jobs.lshd =
      { description = "GNU lshd SSH2 daemon";

        startOn = "started network-interfaces";
        stopOn = "stopping network-interfaces";

        exec = with cfg;
          ''
            # shell script REDACTED
          '';
      };
  };
}
{% endhighlight %}

This basic structure has everything I need to manage a simple SSH reverse forwarding service. It starts and stops with the network, and it executes a shell script.

I started writing an `ssh-phone-home` module using this `lshd` module file as a template, and I made it pretty far before I noticed the following in the NixOS documentation for the `jobs` option:

> This option is a legacy method to define system services, dating from the era where NixOS used Upstart instead of systemd. You should use `systemd.services` instead.

Well great. I guess I should have read that earlier. Oh well; I had to start somewhere, and the conversion to the newer `systemd.services` method is actually rather straightforward.

### The Right Way

The following shows the basic structure of <a href="https://github.com/auntieNeo/nixrc/blob/master/services/ssh-phone-home.nix">ssh-phone-home.nix</a>, once again heavily redacted:

{% highlight nix %}
{ config, lib, pkgs, ... }:
with lib;
let
  inherit (pkgs) openssh;
  cfg = config.services.ssh-phone-home;
in
{
  options = {
    # options REDACTED
  };
  config = mkIf cfg.enable {
    systemd.services.ssh-phone-home =
    { description = "Reverse SSH tunnel as a service.";
      bindsTo = [ "network.target" ];
      serviceConfig = with cfg; { User = cfg.localUser; };
      script = with cfg;  ''
        # shell script REDACTED
      '';
    };
  };
}
{% endhighlight %}

Notice the relatively straightforward differences. The most important difference is that I use `jobs.<name>` instead of `systemd.services.<name>`.

I also replaced the `jobs.<name>.startOn` and `jobs.<name>.stopOn` options with `systemd.services.<name>.bindsTo` which, <a href="http://nixos.org/nixos/manual/#opt-systemd.services._name_.bindsTo">according to the NixOS manual</a>, effectively performs both of those functions.

Finally, I use the `systemd.services.<name>.serviceConfig` option to set the user to run the service as a non-privileged user. This is important because I shouldn't run an SSH client as `root`. This line was taken straight out of <a href="http://blog.kylemanna.com/linux/2014/02/20/ssh-reverse-tunnel-on-linux-with-systemd/">Kyle's script</a>. I just converted it to Nix <a href="http://nixos.org/nixos/manual/#opt-systemd.services._name_.serviceConfig">according to the manual</a>.

## Liberties Taken

I made a few changes to the SSH one-liner from <a href="http://blog.kylemanna.com/linux/2014/02/20/ssh-reverse-tunnel-on-linux-with-systemd/">Kyle's blog post</a>. Here is the shell script that was redacted from the code above:
{% highlight bash %}
${openssh}/bin/ssh -NTC \
    -o ServerAliveInterval=60 -o ExitOnForwardFailure=yes \
    -R ${toString bindPort}:localhost:22 \
    -l ${remoteUser} -p ${toString remotePort} ${remoteHostname}
{% endhighlight %}

I removed the `-o StrictHostKeyChecking=no` option because it seemed a bit dangerous to automatically trust any host keys. Personal experience backs up this kind of thinking, which leads me to my little anecdote on SSH security...

I attended the <a href="https://www.defcon.org/">DEFCON</a> security convention one year, and I specifically remember making sure my `~/.ssh/known_hosts` file had my server's host key in it, thinking I would be safe at the convention as long as I had that. Well, it actually did keep me safe, because every time I tried to connect to my server from the convention hall I would be greeted with `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED`. Someone was performing man in the middle attacks on SSH users, and their passwords were showing up on the <a href="http://www.wallofsheep.com/pages/wall-of-sheep">Wall of Sheep</a>.

I'll leave that option out just in case I ever go back to DEFCON. The only drawback is the service might fail if I don't have the host key, but it's easy to add host keys to `known_hosts`.

I also removed the `-i` option to specify the identity file, but not for any security reasons. The OpenSSH client can usually figure out which identity file to use automatically without the `-i` option.

## Conclusion
I learned my lesson when I didn't check the NixOS manual as soon as I started writing things, but I still feel like `nixpkgs` is a veritable minefield of quirky and deprecated structures. I hope that, in the long run, the older modules get updated to be more consistent with the newer modules. I might even submit a patch for `lshd`, since the transition looks pretty simple.
