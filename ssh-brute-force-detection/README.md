# SSH Brute Force Detection with Wazuh

## What was my goal behind this project

I wanted to learn how a SIEM actually catches an attack, not just read about it. So I set up a small home lab and ran a real SSH brute force attack against a target, then worked through getting my SIEM to actually detect it. This writeup covers what I built and how it all came together. All the problems I ran into along the way, and how I fixed each one, are collected in their own section near the bottom.

## Every step of the way

### Part 1: Getting Logs From My Target to My SIEM

My first goal was to get Metasploitable2's SSH logs forwarding to Wazuh over syslog on UDP 514, so failed logins would actually show up in my SIEM. I added a forwarding line to /etc/syslog.conf on Metasploitable2, telling it to send everything to my Wazuh server:

    *.*                                             @wazuh-ip

Once that was working, I confirmed traffic was actually reaching Wazuh by watching a packet capture on port 514 while triggering a failed login from Kali, and I made sure OPNsense had a firewall rule in place allowing that traffic to cross from Metasploitable2's subnet to Wazuh's subnet.



### Part 2: Confirming Wazuh Was Actually Seeing It

Once packets were arriving, I tested one manual failed SSH login and confirmed it showed up in the Wazuh dashboard under Threat Hunting. It was correctly classified as an authentication failure and even auto tagged with the MITRE ATT&CK technique for password guessing.



### Part 3: Running an Actual Brute Force Attack

I used Hydra from Kali to run a real brute force attack against Metasploitable2's SSH service, using a small list of 10 passwords that did not include the correct one. I wanted to see how Wazuh handled a burst of failures instead of just one.

    hydra -l msfadmin -P passwords.txt ssh://metasploitable-ip

All 10 attempts logged individually as authentication failures under rule 2501, level 5. I learned that Wazuh's default rule 2502, which is named like a brute force detector, is actually just a plain text matching rule with no real counting logic behind it, so I could not rely on it. More on that in the issues section below.

### Part 4: Writing My Own Detection Rule

I wanted real detection that would scale with the size of the attack, so I wrote a custom rule using Wazuh's frequency and timeframe feature, built on top of rule 2501 since it fires reliably on every single failed login.

    <group name="local,syslog,authentication_failed,">
      <rule id="100050" level="12" frequency="6" timeframe="60">
        <if_matched_sid>2501</if_matched_sid>
        <description>SSH brute force attempt: multiple authentication failures from the same source</description>
        <mitre>
          <id>T1110</id>
        </mitre>
      </rule>
    </group>

This rule escalates to a level 12 alert whenever rule 2501 fires 6 or more times within 60 seconds.

After getting the syntax right, I ran Hydra again with the same 10 password list and got two level 12 alerts per run. That checks out with the math: attempts 1 through 6 cross the failure threshold once, then the remaining attempts get partway toward crossing it again. Across the two attack runs I did while testing, the dashboard showed multiple total level 12 alerts, matching what I expected.

### Part 5: Telling a Human Apart From a Script

Getting a rule to fire on repeated failures was one thing. Getting a rule that only fires for something a real person could not have done by hand is a different problem, and it took more thinking than I expected.

SSH itself only allows about 3 failed password attempts before it disconnects you and forces a reconnect. That gave me a real number to reason from instead of guessing. A real person who mistypes their password 3 times in a row is completely normal and not suspicious at all. But a person who gets disconnected after 3 failures, then immediately reconnects and fails 3 more times, all within a minute, is doing something a normal person almost never does.

Based on that, I set my first rule, 100050, to require 6 failures within 60 seconds. That number represents one full round of 3 failures, a reconnect, and a second round of 3 failures, all inside one minute. That is rare for a real person and easy for an automated tool.

I also wanted a second, faster rule that would fire with much higher confidence that the attempt was automated, not just unusual. For that one, rule 100051, I set the threshold much tighter, 5 failures within only 10 seconds. Even with a reconnect in the middle, doing that by hand in 10 seconds is close to impossible for a real person.

    <rule id="100051" level="14" frequency="5" timeframe="10">
      <if_matched_sid>2501</if_matched_sid>
      <description>SSH brute force attempt: rapid automated authentication failures, likely a tool not a person</description>
      <mitre>
        <id>T1110</id>
      </mitre>
    </rule>

One thing I learned along the way, Wazuh's frequency and timeframe counting does not care about SSH connections at all, it only cares about how many times rule 2501 fires inside the time window, no matter how many separate connections those failures came from. A failure from one connection and a failure from a brand new connection a second later both just count toward the same running total. That actually made my threshold choices work better, since it means a real attacker or tool reconnecting quickly after getting kicked still gets caught by the count.

After working through the issue with missing events described below, I settled on these two thresholds and confirmed both rules fire correctly against a live Hydra attack. Rule 100051 tends to fire first, since its tighter 10 second window fills up almost instantly with Hydra's speed, and rule 100050 either fires shortly after or on its own slower 60 second timer depending on how the attempts land.

## Summary

- Syslog forwarding from Metasploitable2 to Wazuh, working end to end
- Cross subnet firewall routing through OPNsense, working end to end
- A custom correlation rule, 100050, that fires on 6 failures within 60 seconds, built, debugged, and confirmed against a live attack
- A second, faster rule, 100051, that fires on 5 failures within 10 seconds, a high confidence tier that catches attack speed no real person could match by hand

## What I would want to do next

- Add same source IP tracking back into my rules once I set up a proper decoder that extracts a source IP from the plain syslog failure events, so the rules only correlate failures coming from one attacker instead of any matching failures lab wide.
- Tie an active response to rule 100051, like a temporary firewall block, so it moves from just detecting to actually responding.
- Run more repeated tests to see how consistent the syslog collapsing behavior described below actually is, since that would tell me whether my safety margin on rule 100051 is generous enough or needs to be wider.

## Issues I ran into and how I fixed them

My syslog forwarding line was written wrong at first. It was missing the star dot star part at the start and the @ symbol before the IP address. Since it did not match the syntax syslog expects, syslogd just ignored it completely, with no error message and nothing showing up in a packet capture on the Wazuh side. After fixing the line, I made a habit of checking with grep '@' /etc/syslog.conf to confirm the line actually saved, since an edit that fails to save silently looks exactly the same as an edit that was never made.

My firewall was silently blocking the traffic even after the syslog line was fixed. I had already created a floating firewall rule to allow UDP 514 from Metasploitable2 to Wazuh, with the correct source and destination aliases, but I still got zero packets in tcpdump. I found the real cause by watching OPNsense's live firewall log while triggering a failed login, which showed the traffic hitting a default Block LAN to WAN rule before my own rule ever got a chance to run. My floating rule had the wrong interface and direction set. It needed to be scoped to the LAN interface, direction in, with Quick checked so it gets evaluated before the block rule further down. A firewall rule can have the right source and destination and still fail completely if the interface or direction is wrong, and it will fail silently.

The default rule that looked like brute force detection was not really detecting brute force. During my first Hydra run, only one alert jumped up to a higher severity, rule 2502, level 10, described as user missed the password more than one time. I dug into what rule 2502 actually does and found out it is just a plain text matching rule. It only fires if a log line literally contains the phrase more authentication failures or repeated login failures, it is not counting anything or tracking time at all. The one alert it produced only happened because Linux's syslog daemon collapsed several identical repeated log lines into a single summary line, and that summary line happened to contain wording that matched. A rule's name can say brute force while its actual logic does something completely different, so I learned to actually check what a rule does instead of trusting its description.

Frequency and timeframe have to be attributes on the rule tag itself, not their own separate tags underneath. My first version of rule 100050 had them as nested tags and Wazuh rejected the whole file with an invalid option error.

Wazuh's default local_rules.xml file ships with an example rule already using id 100001. I accidentally reused that same ID for my rule, which caused a silent conflict and meant nothing ever fired. Changing it to an unused ID, 100050, fixed it.

At one point I edited the wrong file entirely, a default Wazuh ruleset file instead of local_rules.xml. Default files get overwritten on updates, so anything custom needs to live in local_rules.xml specifically.

XML attributes cannot have a space before the equals sign. I once wrote the level attribute with a stray space before its equals sign, and Wazuh threw an error pointing at the exact line, attribute level has no value. Reading that error message told me exactly what to fix.

When I first tested rule 100051, it did not fire even though my Hydra attack was clearly fast enough. I dug through the dashboard and found the real reason. Out of 10 total password attempts, only 9 showed up as rule 2501. The 10th one got absorbed into rule 2502 instead, because syslog collapsed a couple of identical repeated log lines into a single summary line that happened to match rule 2502's specific wording instead of logging as a normal 2501 failure. That meant my real count of qualifying 2501 events was one short of what I needed, even though the actual attack sent a full 10 attempts. This was not a timing problem, it was a counting problem caused by syslog's own behavior of collapsing duplicate messages during a fast burst. Knowing that, I lowered rule 100051's threshold from 10 down to 5, which still represents something no real person could do by hand in 10 seconds, but also builds in enough of a safety margin that losing a message or two to syslog's collapsing does not stop the rule from firing when it should.
