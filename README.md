# Cross-Domain Conversion Tracking Setup: The Unseen Data Black Hole

Somewhere between 30 and **50 percent** of [conversion](/conversion-api)s in a multi-domain funnel lose their [attribution](/resources/marketing-attribution-models-from-last-click-to-data-driven) source. Not because someone forgot to configure anything. Because the funnel crosses a domain boundary, and a domain boundary is where tracking quietly goes to die.

I have debugged this on more checkout-on-a-separate-domain setups than I want to remember, and the pattern is always the same. The store owner did the [GA4](/resources/ga4-server-side-implementation-guide) cross-domain config. They added the second domain to the linker. They tested it once, saw a session survive the jump, and called it done. Then months later they notice their own domain showing up as a referral source and a strange spike in "new" users, and they think it is a small bug.

It is not a small bug. It is a black hole. This is not a "fix your cross-domain config" post - those exist and they are fine as far as they go. This is a post about why a perfectly correct config still leaks, where the leaked data goes, and what it does to your ad spend when it gets there.

The short version: cross-domain tracking depends on a parameter being passed in a browser, by a script, at the exact moment a user moves between domains. Every one of those things can fail. When it fails, the session does not error out. It splits in two. And the orphaned half still reaches Google and [Meta](/meta-conversion-api) wearing a costume. DataCops fixes this at the architecture level, by not depending on that fragile browser handoff in the first place. More on that after you see the gap.







## Quick stuff people keep asking

**How do I set up cross-domain tracking in [GA4](/alternative/ga4-alternative)?** In the GA4 data stream, open Configure tag settings, then Configure your domains, and list every domain in the funnel. GA4 then appends a linker parameter to outbound links between those domains so the client ID carries across. That is the whole official setup. It is also the whole official fragility.

**Why is cross-domain tracking not working in Google Analytics 4?** Usually the linker parameter never made it across. The link was opened in a new context, a redirect stripped the query string, the script had not loaded when the click happened, or the destination domain was not in the configured list. The session breaks and GA4 starts a fresh one.

**What is the GA4 linker parameter?** It is the `_gl` value GA4 sticks onto links between your domains. It carries the client ID so analytics treats domain A and domain B as one journey. If `_gl` does not arrive intact, the journey becomes two journeys.

**Why do I see my own domain as a referral source in GA4?** Classic symptom of a broken handoff. The user crossed from your site to your checkout domain, the client ID did not travel, so GA4 saw a brand-new visitor arriving from your first domain. Your own site became its own traffic source. That is a session that split.

**How do cross-domain cookies work in analytics?** They mostly do not, and that is the root issue. Cookies are scoped per domain. A cookie set on domain A is invisible to domain B. The linker parameter exists precisely because cookies cannot cross. So the whole mechanism leans on a URL parameter surviving a browser navigation, which is a weaker guarantee than people assume.

**Does cross-domain tracking affect conversion attribution?** Directly. When the session splits, the conversion lands on a session with no memory of the campaign that drove it. The sale still happened. The credit for it evaporated, or got handed to "direct".

**How do I track conversions across a checkout subdomain?** A true subdomain - checkout dot yourstore dot com - is far easier, because a cookie can be scoped to the parent domain and shared. A separate domain entirely cannot do that. If you can keep checkout on a subdomain, do. If it is a different domain, you are in cross-domain territory and all the fragility applies.

**Why does GA4 show inflated new user counts?** Every split session mints a phantom new user. The same person, counted twice, the second copy labeled "new". Multiply that across a funnel and your new-user number is structurally inflated and your returning-user number is structurally deflated.

## The black hole: where the attribution actually goes

Here is the part the fix guides skip. When cross-domain tracking fails, the data is not lost. Lost would be cleaner. The data survives, it just survives wrong.

The session splits at the boundary. The first half remembers the campaign - the [Google Ads](/google-conversion-api) click, the Meta ad, the UTM. The second half, the half where the purchase happens, remembers nothing. So the conversion gets recorded against a session whose source is your own domain, or "direct", or "(none)".

That second-half session still gets reported. It still flows to GA4. And through your conversion connections, a version of it still reaches Google and Meta. Now think about what that means. The ad platforms receive a conversion with no campaign attached, or attributed to the wrong source entirely. From their side, that looks like a sale that happened without their ad. So the campaign that genuinely drove it gets under-credited, and the platform's optimization engine learns that the ad underperformed.

It did not underperform. The handoff broke. But the algorithm cannot tell the difference between "this ad did not work" and "the tracking lost the thread", so it does the rational thing with bad input. It pulls budget away from the campaign that actually worked.

That is the black hole. Revenue you genuinely earned, mis-filed, and then used as evidence against the campaign that earned it.

And it compounds. The mis-attributed conversions become the training set. Google and Meta study which conversions came from where, and adjust. Feed them a stream where 30 to **50 percent** of conversions have the wrong source, and you are not just losing reporting accuracy. You are actively teaching the optimization engine a false map of what drives your sales. Garbage in, garbage optimized, budget moved in the wrong direction.

Picture a honeypot test someone ran on a signup flow - three thousand signups, seventy-seven percent fraud, 650 accounts traced to one device. That is the visceral version of "the data was wrong and the system believed it anyway". Cross-domain attribution loss is the quieter version. No fraud, no dramatic number. Just a steady, invisible mis-filing of real money, and an algorithm dutifully optimizing against it.

## Why correct config still is not enough

Get the config perfect and you still have a structural exposure. The mechanism itself is fragile.

It depends on a third-party script being loaded and ready at the moment of the click. On a single-page app, route transitions and re-renders create race conditions where the handoff happens before the tracking is ready. It depends on a URL parameter surviving the navigation - and redirects, link wrappers, new browsing contexts and parameter stripping all eat it. It depends on the user's browser cooperating, and privacy browsers and tracking protection do not.

You cannot configure your way out of a design that assumes a perfect browser handoff every single time. The handoff will fail some of the time. The only question is whether your tracking degrades gracefully or splits a session and ships a phantom.

The architectural answer is to stop depending on the browser handoff. A [first-party](/first-party-consent-manager-platform) setup, running on your own subdomain, identifies and stitches the journey server-side instead of betting everything on a parameter surviving a click. The conversion is tied to the journey before it leaves your infrastructure, not reconstructed afterward from whatever fragments the browser managed to keep.

That is what DataCops does. It runs first-party, stitches the funnel server-side, filters [bot](/fraud-traffic-validation) traffic at ingestion against a 361.8 billion-plus IP database so phantom and automated sessions are not counted as real users, and forwards clean conversion data via CAPI to Meta, Google, TikTok and LinkedIn. It also keeps two data tiers separate at the source - anonymous session analytics flow unconditionally, identifiable data is handled on its own track. The conversion that reaches your ad platforms carries the source it actually came from.

Honest limitations: DataCops is a newer brand than the household analytics names, and SOC 2 Type II is in progress, not complete. If you need that certificate today, plan around the timing. What it does now is close the black hole - and the black hole is the expensive part.

## Decision guide

**Single domain, no separate checkout.** You do not have a cross-domain problem. Do not invent one. Skip this entirely.

**Checkout on a true subdomain.** Scope your cookie to the parent domain, confirm sessions survive the jump, and you are largely fine. Verify the linker anyway, but a subdomain is the easy case.

**Checkout on a separate domain.** This is the real exposure. Configure GA4 cross-domain, then accept that config alone leaks. Move to a first-party setup that stitches the journey server-side.

**Multi-domain funnel and your ROAS does not match what you feel is working.** That mismatch is the black hole talking. Audit how many conversions arrive as "direct" or self-referral. That number is your leak.

**You sell into the EU.** Keep anonymous analytics flowing across domains unconditionally - that is always legal. Gate identifiable data behind consent. Separate the tiers at the source rather than mixing them and sorting later.

## You are not losing data. You are mis-filing money.

The mistake almost everyone makes with cross-domain tracking is treating it as a setup task with a finish line. Configure the domains, see one session survive, check the box, never look again. But there is no finish line, because the mechanism leaks by design every time the browser handoff stumbles, and it leaks silently.

The conversions are not vanishing. They are landing in the wrong file, getting reported to Google and Meta with the wrong source, and being used as evidence to defund the campaigns that actually earned them.

So pull last month's GA4 report. Look at how many conversions are attributed to "direct" or to your own domain as a referral. Be honest about how many of those were really direct. Whatever that gap is, that is the money you earned and then told your ad platforms to ignore. How big is your black hole?

---

Research by [DataCops](https://www.joindatacops.com) — first-party tracking, consent infrastructure, fraud prevention, and server-side CAPI for Meta, Google, TikTok, and LinkedIn.
