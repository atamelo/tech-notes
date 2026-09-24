# Computing Connection Degrees in Social Networks (Claude Opus 4.6)

*A conversation with Claude Opus 4.6: how LinkedIn-style networks show 1st/2nd/3rd-degree labels without real-time BFS or all-pairs precomputation - per-user neighborhoods, set intersections, and how the 2-hop sets get built, updated and stored.*

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Can you explain something to me?
<br><br>
Specifically, how social networks like LinkedIn do the following.
<br><br>
Afaik, connections between users are stored as a graph (assuming, in some kind of a graph db), which makes sense.
<br><br>
But when I see another user (e.g. in a search, or as suggested connection) the system displays whether that user is 1st circle, 2nd, 3rd, etc.
<br><br>
I do not believe that the system does something like BFS in real time to show that info.. I also do not think that it's realistic to pre-compute relationship "level" (i.e. 2nd circle, 3rd and so on) between all possible pairs of existing users.
<br><br>
So, how does it do that, really?
</td></tr>
</table>

Great question - and your intuitions are right on both counts. Real-time BFS across hundreds of millions of nodes would be too slow, and precomputing all pairs is combinatorially impossible (LinkedIn has ~1B users; that's ~10^18 pairs). So what actually happens sits in between those extremes.

The core trick is that LinkedIn doesn't need to compute the shortest path between arbitrary pairs. It only needs to answer the question *from the perspective of the currently logged-in user*. That changes the problem entirely.

**The key idea: precompute per-user, not per-pair.**

When you log in, LinkedIn can materialize (or pull from cache) your **1st-degree set** - the list of your direct connections. For most users this is a few hundred to maybe a few thousand IDs. That's tiny and fits easily in memory.

Your **2nd-degree set** is the union of your connections' connection lists. This is bigger (potentially millions), but it's still bounded and can be computed with a single fan-out: for each of your connections, look up *their* connection list, and merge. LinkedIn almost certainly doesn't do this naively for every page load - instead they precompute and cache it, refreshing periodically or on connection-change events.

Once you have sets for degrees 1 and 2, answering "what degree is this person?" for any profile you encounter is just a set membership check: is this user ID in my 1st-degree set? My 2nd-degree set? If neither, they're 3rd+ (and LinkedIn generally stops distinguishing at 3rd).

**How it scales in practice:**

The precomputation is done offline, not on every request. LinkedIn has talked publicly about systems like this. They maintain a graph service that, for each user, stores adjacency lists (your connections). On top of that, they build what's essentially a **second-degree index** - a precomputed mapping that tells you all the people reachable within 2 hops. This gets rebuilt periodically (or incrementally updated when connections change). The result is stored in a fast key-value store or distributed cache.

When you view a search result page with 10 profiles, the system does 10 lookups against your precomputed index. That's microseconds, not graph traversal.

**A few additional details that make this practical:**

First, 3rd degree doesn't need full precomputation either. If someone isn't in your 1st or 2nd degree set, LinkedIn can check whether *any* of your 2nd-degree connections have them as a 1st-degree connection. This is still a set intersection problem, not a full BFS. And in many UI contexts, LinkedIn just labels anything beyond 2nd as "3rd+" without confirming exact distance.

Second, they use a technique sometimes called **bidirectional bounded BFS** for cases where they do need real-time computation - you expand one hop from the viewer, one hop from the target, and check for overlap. With depth limited to 3 total hops and a lot of pruning, this is tractable even for large graphs.

Third, LinkedIn built a dedicated system for this called **Galene** (for search indexing) and has graph infrastructure that partitions adjacency lists across a cluster, keeping hot data (your connections and their connections) in memory. They also use bloom filters as a compact probabilistic structure to quickly answer "is X in my 2nd-degree set?" without storing the full set - a bloom filter for a few million IDs only takes a couple megabytes.

So the short answer: it's **per-user precomputation of nearby neighborhoods** (1st and 2nd degree sets), stored in fast caches, checked via set membership at query time. Not real-time BFS, not all-pairs precomputation - the sweet spot in between.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; So for:
<br><br>
<code>N(A) = {A's direct connections}</code>
<br><br>
The algorithm is the following:
<pre style="background: rgba(255, 255, 255, 0.12); color: #ffffff; padding: 10px 14px; border-radius: 10px; margin: 10px 0;">
if B in N(A):
    1st degree
else if N(A) ∩ N(B) is non-empty:
    2nd degree
else if some neighbor of B is in A's 2-hop set:
    3rd degree
else:
    out of network
</pre>
Right?
</td></tr>
</table>

Exactly right. And it's worth noticing *why* each step is formulated the way it is - each one avoids materializing something expensive.

The 1st-degree check is trivial - just a lookup in A's adjacency list.

The 2nd-degree check is the clever part. You *could* check `B ∈ N(N(A))`, meaning "is B in A's full 2-hop set?" But that requires materializing or indexing that entire 2-hop set. The intersection formulation `N(A) ∩ N(B)` is equivalent but operationally nicer - you only need the two adjacency lists, which you already have. And as a bonus, the members of that intersection are the "mutual connections" LinkedIn shows you on the profile ("You and B share 12 connections"). So the same computation serves double duty.

The 3rd-degree check follows the same pattern. You *could* try to build a 3-hop set from A, but that's enormous. Instead you flip it: take B's neighbors (small list), and check each against A's 2-hop set. That's where the precomputed index or bloom filter earns its keep - you're doing a handful of membership checks against a structure you've already built, rather than expanding the frontier further outward from A.

So each step is designed to keep the work proportional to the *smaller* side of the query rather than exploding outward into the full graph.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; But for the:
<br><br>
<code>second[A] = users reachable from A in 2 hops</code>
<br><br>
having the 2-hop reachable set for user A is still required. How exactly do they go about extracting 2-hop reachable users for A?
</td></tr>
</table>

Right - you can't avoid building it, you can only choose *when* and *how* to build it. A few practical approaches:

**Offline batch precomputation.** Run a periodic job (MapReduce, Spark, etc.) that iterates over all users. For each user A, look up N(A), then for each member C of N(A), look up N(C), union the results, and write the output to a distributed key-value store. This is embarrassingly parallel - each user's 2-hop set is independent. LinkedIn processes this on the order of hours, not real-time. The result sits in something like Voldemort or a similar distributed store, keyed by user ID.

**Incremental updates.** You don't recompute from scratch every time someone adds a connection. If user X connects with user Y, you know exactly whose 2-hop sets change: the members of N(X) gain Y's connections, and the members of N(Y) gain X's connections. So you push targeted updates to those affected sets. In practice this is done via an event stream - connection changes produce events, a consumer updates the relevant cached sets.

**Lazy materialization with caching.** For less active users, you might not maintain the 2-hop set at all until they log in. On login (or first search), you compute it on the fly - fan out across N(A), gather all their adjacency lists, union them. For a user with 500 connections, each having ~500 connections, that's 500 adjacency list lookups and a union of maybe 200K IDs. With adjacency lists sitting in a memory-backed store, that's doable in tens of milliseconds. Then you cache the result with a TTL, so subsequent page views during the session don't recompute.

**Compact representations.** The full 2-hop set for an active user might be millions of IDs. Storing that verbatim for every user is expensive. So in practice they often use bloom filters - a probabilistic structure that can answer "is X in A's 2-hop set?" with a small false-positive rate and zero false negatives. A bloom filter for a few million entries fits in a couple megabytes. You accept the occasional "3rd degree" user being misclassified as "2nd degree" - a harmless error in this context.

So in short: batch-compute it offline for active users, store it compactly (bloom filters or compressed ID sets), keep it warm in cache, and patch it incrementally as the graph changes. The key insight is that it doesn't need to be perfectly fresh - if your coworker added a connection 10 minutes ago and your 2-hop set hasn't caught up yet, nobody notices.
