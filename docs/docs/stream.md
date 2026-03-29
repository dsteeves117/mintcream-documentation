## 1. Default Stream page

### `GET /stream/`

**When to use:** Use this endpoint to render a stream containing all entries registered on this node which are visible to the current user.

**What is rendered:**
```
Public entries, unlisted entries by users you follow, friends-only entries by users you are friends with, entries that you are the author of,
and the 2 most recent comments on visible entries, if applicable.
Deleted entries will not be rendered regardless of whether the above conditions are met.
If a user is not signed in, only public entries will be rendered.
Entries will be rendered in order of most recently posted.
```

**Authentication:** None required (public endpoint).

**Pagination:** No

#### Response
```
200 OK
Content-Type: text/html
```

**URL** `source/stream/`

#### Examples

**Show stream** `GET /stream/`

**Response:**
```
200 OK
Content-Type: text/html
```

**HTML body rendered when no visible entries exist** 
```
<h1 class="section-title" style="margin-top:0;">Stream</h1>
<p style="opacity:.6;">Nothing in your stream yet. Follow some people or make a post!</p>
```

**Example HTML body rendered for 2 entries, where the first has 2 comments and the second has no comments**
```
<h1 class="section-title" style="margin-top:0;">Stream</h1>

<div class="card"  style="margin-bottom:0.4rem;" >
    <div class="card-meta card-pfp">
      
        <div class="profile-avatar-placeholder small-circle">J</div>
      
      <div>
        <a href="/authors/3cf6c227-d073-494c-8ee6-757665c069dd/">JerryJohn57</a>
        &nbsp;·&nbsp; March 1, 2026, 9:42 a.m.
      </div>
    </div>
    <h2 class="card-title">
      <a class="card-link" href="/authors/3cf6c227-d073-494c-8ee6-757665c069dd/entries/b0b26697-2b7d-49b3-b5e3-cb203e6341a2/">PUBLIC JERRYPOST</a>
    </h2>
    <p class="card-desc">this post is public and made by jerryjohn</p>
    

  <pre style="white-space: pre-wrap;">HELLO I HAVE COME TO MAKE AN ANNOUNCEMENT</pre>

    </div>
  
  <div style="margin-bottom: 1.25rem;">
    
      <div class="card" style="margin-bottom:0.4rem;">
        <div class="card-meta card-pfp">
          
            <img class="small-circle" src="https://th.bing.com/th/id/OIP.tFUowpd0Xuqjf5yShTbMKQHaL5?w=118&amp;h=190&amp;c=7&amp;r=0&amp;o=7&amp;dpr=1.3&amp;pid=1.7&amp;rm=3" alt="SpaceLobster21">
          
          <div>
            <a href="/authors/60ccfbfe-b409-418b-82aa-71ed3d872ecb/">SpaceLobster21</a>
            &nbsp;·&nbsp; March 2, 2026, 8:58 a.m.
          </div>
        </div>
        <p style="margin:.25rem 0 0; white-space:pre-wrap;">I&#x27;M BEGGING YOUUUU</p>
      </div>
    
      <div class="card" style="margin-bottom:0.4rem;">
        <div class="card-meta card-pfp">
          
            <img class="small-circle" src="https://th.bing.com/th/id/OIP.tFUowpd0Xuqjf5yShTbMKQHaL5?w=118&amp;h=190&amp;c=7&amp;r=0&amp;o=7&amp;dpr=1.3&amp;pid=1.7&amp;rm=3" alt="SpaceLobster21">
          
          <div>
            <a href="/authors/60ccfbfe-b409-418b-82aa-71ed3d872ecb/">SpaceLobster21</a>
            &nbsp;·&nbsp; March 2, 2026, 8:58 a.m.
          </div>
        </div>
        <p style="margin:.25rem 0 0; white-space:pre-wrap;">JERRYJOHN PLEASE NOTICE ME</p>
      </div>
    
  </div>

  <div class="card" >
    <div class="card-meta card-pfp">
      
        <div class="profile-avatar-placeholder small-circle">J</div>
      
      <div>
        <a href="/authors/3cf6c227-d073-494c-8ee6-757665c069dd/">JerryJohn57</a>
        &nbsp;·&nbsp; March 1, 2026, 8:00 a.m.
      </div>
    </div>
    <h2 class="card-title">
      <a class="card-link" href="/authors/3cf6c227-d073-494c-8ee6-757665c069dd/entries/c2deb02f-a84e-4f8d-9e7c-257563370699/">POTATO</a>
    </h2>
    <p class="card-desc">A post about a potato</p>
    

  <pre style="white-space: pre-wrap;">I really want a potato right now</pre>

  </div>
```

**Example scenario**
```
Suppose we have:
Author A:
  Posted entries:
    A_public
    A_unlisted
    A_friendsOnly
    A_deleted
  Has commented "A_comment" on B_public

Author B:
  Posted entries:
    B_public
    B_unlisted
    B_friendsOnly
    B_deleted
  Is following Author C (friends with C)
  Has commented "B_comment1" on B_public
  Has commented "B_comment2" on C_unlisted

Author C:
  Posted entries:
    C_public
    C_unlisted
    C_friendsOnly
    C_deleted
  Is following Author A and Author B (friends with B)
  Has commented "C_comment" on B_public

---

Then, assuming all entries have the visibility implied by their name (i.e. A_friendsOnly has Friends Only visibility) and the entries were posted in the order listed (i.e. A_public being posted is the oldest action and C_comment being posted is the most recent action):

---

User A's stream:

  Stream

  C_public

  B_public
  | C_comment
  | B_comment1

  A_friendsOnly
  
  A_unlisted

  A_public

---

User B's stream:

  Stream

  C_friendsOnly

  C_unlisted
  | B_comment2

  C_public

  B_friendsOnly

  B_unlisted

  B_public
  | C_comment
  | B_comment1

  A_public

---

User C's stream:

  Stream

  C_friendsOnly

  C_unlisted
  | B_comment2

  C_public

  B_friendsOnly

  B_unlisted

  B_public
  | C_comment
  | B_comment1

  A_public

  A_unlisted

```




## 2. Following Filter

### `GET /stream/?filter=following`

**When to use:** Use this endpoint to render a stream containing all entries registered on this node which were created by authors the current user follows.

**What is rendered:**
```
Public entries by users you follow, unlisted entries by users you follow, friends-only entries by users you are friends with,
and the 2 most recent comments on visible entries, if applicable.
Deleted entries will not be rendered regardless of whether the above conditions are met.
If a user is not signed in, filter will not be applied and the result will only show public entries.
Entries will be rendered in order of most recently posted.
```

**Authentication:** None required (public endpoint).

**Pagination:** No

#### Response
```
200 OK
Content-Type: text/html
```

**URL** `source/stream/?filter=following`

#### Examples

**Show stream with filter applied** `GET /stream/?filter=following`

**Response:**
```
200 OK
Content-Type: text/html
```

**Example scenario (same data as before, but with filter applied)**
```
Suppose we have:
Author A:
  Posted entries:
    A_public
    A_unlisted
    A_friendsOnly
    A_deleted
  Has commented "A_comment" on B_public

Author B:
  Posted entries:
    B_public
    B_unlisted
    B_friendsOnly
    B_deleted
  Is following Author C (friends with C)
  Has commented "B_comment1" on B_public
  Has commented "B_comment2" on C_unlisted

Author C:
  Posted entries:
    C_public
    C_unlisted
    C_friendsOnly
    C_deleted
  Is following Author A and Author B (friends with B)
  Has commented "C_comment" on B_public

---

Then, assuming all entries have the visibility implied by their name (i.e. A_friendsOnly has Friends Only visibility) and the entries were posted in the order listed (i.e. A_public being posted is the oldest action and C_comment being posted is the most recent action):

---

User A's stream:

  Stream

  Nothing in your stream yet. Follow some people or make a post!

---

User B's stream:

  Stream

  C_friendsOnly

  C_unlisted
  | B_comment2

  C_public

---

User C's stream:

  Stream

  B_friendsOnly

  B_unlisted

  B_public
  | C_comment
  | B_comment1

  A_public

  A_unlisted

```




## 3. Friends_only Filter

### `GET /stream/?filter=friends_only`

**When to use:** Use this endpoint to render a stream containing all entries registered on this node which were created by authors the current user is friends with.

**What is rendered:**
```
Public entries by users you are friends with, unlisted entries by users you are friends with, friends-only entries by users you are friends with,
and the 2 most recent comments on visible entries, if applicable.
Deleted entries will not be rendered regardless of whether the above conditions are met.
If a user is not signed in, filter will not be applied and the result will only show public entries.
Entries will be rendered in order of most recently posted.
```

**Authentication:** None required (public endpoint).

**Pagination:** No

#### Response
```
200 OK
Content-Type: text/html
```

**URL** `source/stream/?filter=friends_only`

#### Examples

**Show stream with filter applied** `GET /stream/?filter=friends_only`

**Response:**
```
200 OK
Content-Type: text/html
```

**Example scenario (same data as before, but with filter applied)**
```
Suppose we have:
Author A:
  Posted entries:
    A_public
    A_unlisted
    A_friendsOnly
    A_deleted
  Has commented "A_comment" on B_public

Author B:
  Posted entries:
    B_public
    B_unlisted
    B_friendsOnly
    B_deleted
  Is following Author C (friends with C)
  Has commented "B_comment1" on B_public
  Has commented "B_comment2" on C_unlisted

Author C:
  Posted entries:
    C_public
    C_unlisted
    C_friendsOnly
    C_deleted
  Is following Author A and Author B (friends with B)
  Has commented "C_comment" on B_public

---

Then, assuming all entries have the visibility implied by their name (i.e. A_friendsOnly has Friends Only visibility) and the entries were posted in the order listed (i.e. A_public being posted is the oldest action and C_comment being posted is the most recent action):

---

User A's stream:

  Stream

  Nothing in your stream yet. Follow some people or make a post!

---

User B's stream:

  Stream

  C_friendsOnly

  C_unlisted
  | B_comment2

  C_public

---

User C's stream:

  Stream

  B_friendsOnly

  B_unlisted

  B_public
  | C_comment
  | B_comment1

```