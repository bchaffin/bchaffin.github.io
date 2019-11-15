---
layout: post
title:  "A Smart(er) Thermostat"
date:   2019-11-13
---

My house is heated by an air-source heat pump, installed in
2008 -- the Mitsubishi PUMY-P48, with indoor air handler
PEFY-P54. These units appear to be discontinued, but it's an efficient
model, and can produce heat even with very low outdoor temperatures --
in fact, unlike most heat pump installations, we have no backup
electric-resistance heater for when it's very cold outside.

It generally works well, but its heat output varies greatly depending
on the outdoor temperature. When it's in the 40s outside, it might run
for 30-40 minutes in a typical cycle. But when it's in the 20s (which
is rare here in Portland, OR), a cycle might take 2-3 hours. Which
means with a typical programmable thermostat, it's hard to know when
to turn on the heat. And in a really cold snap, it needs to run almost
continuously -- if I let the temperature of the house drop too much
overnight, it might take all day or even longer to warm it up again.

Also, 3 years ago our electric utility started a time-of-use pilot
program (recently ended), which charged more during peak hours of the
morning and evening.

For both of these reasons, I really wanted to be able to predict how
long it would take to warm up the house, and turn on the heat so as to
be done heating at a certain time.

## The Ecobee

A "smart thermostat" seemed like the way to go. The two main models on
the market at the time (late 2015) were the [Ecobee
3](https://www.ecobee.com/ecobee3/) and the
[Nest](https://nest.com/). They both offered a feature which did
exactly what I wanted -- you set the time you want the house to be
warm, instead of the time the heat should turn on -- but the Ecobee
claimed to have a more flexible solution, so that's what I chose.

#### The hookup

The first challenge was even connecting it to the heat
pump. Mitsubishis of that era came with their own thermostat, which
used a proprietary controller interface that communicates via encoded
signals over 2 wires -- not the standard simple 3-wire or 4-wire
thermostat connection.

After a lot of research and hacking, I found an internal interface in
the air handler's circuit board which allows for some kind of override
-- I think intended for a whole-site shutoff at night, in a commercial
setting with many independent units. Unfortunately I did not document
all the details at the time, but the solution involved a 24VAC
transformer, a relay, and some custom wiring. The Mitsubishi
controller is still required, and thinks it's running the show -- but
the Ecobee controls the master override, and can just switch the whole
system on and off. The following summer some additional trickery was
needed to get it to work with air conditioning, which in this setup
requires the Ecobee to do the same thing as for heating; but of course
the Ecobee wants to use different wires for AC.

I think it's very unlikely that this is a problem anyone will ever
have to solve again; around 2012 Mitsubishi changed to allow control
by a standard interface. But if for some reason you are trying to use
a smart thermostate to control a pre-2012 Mitsubshi heat pump, and by
some miracle you actually find this page, you are welcome to contact
me and I'll reverse-engineer what I have.

#### Not so smart

Unfortunately, the Ecobee's "Smart Recovery" feature was a complete
failure. Occasionally it turned on the heat at seemingly arbitrary
times, but most of the time it just didn't do anything at all. (It's
possible they have improved the feature since 2016 -- I haven't tried
it again.)

Otherwise, it has worked fine. The interface is easy and having access
and control of the thermostat from a phone is often useful.

## Fine, I'll do it myself

But I still wanted predictive preheating. The Ecobee offers a [pretty
well-documented
API](https://www.ecobee.com/home/developer/api/examples/ex1.shtml) for
external applications. And they allow 85000 queries a month for free,
which is pretty generous -- about 2 per minute.

Ecobee samples the state of your thermostat every 5 minutes, and
appears to store the data indefinitely. (Hmm.) By the time I started
work on this, I had had the Ecobee for just about a year. So I
downloaded the full year's history, which included indoor and outdoor
temperatures, thermostat settings, and whether the heat was currently
running.

For every time that the heat cycled on and off over the course of that
year, I computed how long it took to raise the house 1 degree, and
split all those rates out by the outdoor temperature at the time. (I
discarded the top and bottom 10% of the measurements to reduce noise.)
This gave me my "model" of the house, which is just a prediction of
how quickly the indoor temperature will rise depending on the outdoor
temperature. For example, at 55 degrees outside, it takes 20 minutes
to gain 1 degree; at 40 it takes 26 minutes, and at 25 it takes 57
minutes.

A script runs every 12 minutes on my home computer. It gets the
current state of the Ecobee, which includes the indoor temperature and
the schedule for the next 24 hours. It parses the XML output using
[Cpanel::JSON:XS](https://metacpan.org/pod/Cpanel::JSON::XS) and
[DateTime::Format::ISO8601](https://metacpan.org/pod/DateTime::Format::ISO8601),
and looks to see if the temperature setting is scheduled to go up
within the next 8 hours -- this allows me to still control things by
changing the thermostat's schedule, rather than having to modify the
script on my computer.

If a temperature increase is scheduled, it gets the current outdoor
conditions from [OpenWeatherMap](https://openweathermap.org/), and
uses the model to predict how long it would take to get to the
scheduled temperature from the current indoor temperature. Then at the
right time, it overrides the current temperature setting to match the
scheduled increase.

Computing the model was a one-time analysis. It might be better to
continuously integrate recent behavior, which would allow it to adapt
to any changes in the house or heating system performance. But
actually I kind of like that if something changes unexpectedly -- say
the output of the heat pump drops off, or someone left a basement
window open -- I'll notice because the heat will no longer finish at
the right time.

In practice the system works really well, and the static model hasn't
been a problem. Its predictions of how long the heat will need to run
are typically accurate to within 10 minutes or so. It's nice to always
come home to a warm house, regardless of the weather, and it's
satisfying every time I hear the heat switch off right on schedule.
