# WordPress database clean up queries
- [Orphan rows](#orphan-rows)
	- [wp_posts -> wp_posts (parent/child)](#wp_posts---wp_posts-parentchild)
	- [wp_postmeta -> wp_posts](#wp_postmeta---wp_posts)
	- [wp_term_taxonomy -> wp_terms](#wp_term_taxonomy---wp_terms)
	- [wp_term_relationships -> wp_term_taxonomy](#wp_term_relationships---wp_term_taxonomy)
	- [wp_usermeta -> wp_users](#wp_usermeta---wp_users)
	- [wp_posts -> wp_users](#wp_posts---wp_users)
- [Other](#other)
	- [wp_postmeta dupes](#wp_postmeta-dupes)
	- [wp_postmeta dupes #2](#wp_postmeta-dupes-2)
	- [wp_postmeta missing](#wp_postmeta-missing)
	- [wp_postmeta '_edit_lock' and  '_edit_last' rows](#wp_postmeta-_edit_lock-and--_edit_last-rows)
	- [wp_options '_transient_' rows](#wp_options-transient-rows)
	- [wp_posts revisions](#wp_posts-revisions)
- [Orphaned Term Relationships](#orphaned-term-relationships)

## Orphan rows
Since WordPress uses MyISAM for it's storage engine, we don't get foreign keys relationships as offered by InnoDB/etc. - thus orphan rows can show themselves over time.

### wp_posts -> wp_posts (parent/child)

```sql
SELECT * FROM wp_posts
LEFT JOIN wp_posts child ON (wp_posts.post_parent = child.ID)
WHERE (wp_posts.post_parent <> 0) AND (child.ID IS NULL)

DELETE wp_posts FROM wp_posts
LEFT JOIN wp_posts child ON (wp_posts.post_parent = child.ID)
WHERE (wp_posts.post_parent <> 0) AND (child.ID IS NULL)
```

### wp_postmeta -> wp_posts

```sql
SELECT * FROM wp_postmeta
LEFT JOIN wp_posts ON (wp_postmeta.post_id = wp_posts.ID)
WHERE (wp_posts.ID IS NULL)

DELETE wp_postmeta FROM wp_postmeta
LEFT JOIN wp_posts ON (wp_postmeta.post_id = wp_posts.ID)
WHERE (wp_posts.ID IS NULL)
```

### empty post metas
WP will return false anyway if post meta not exists.

```sql
DELETE FROM wp_postmeta WHERE meta_value = NULL OR meta_value = ""
```



### wp_term_taxonomy -> wp_terms

```sql
SELECT * FROM wp_term_taxonomy
LEFT JOIN wp_terms ON (wp_term_taxonomy.term_id = wp_terms.term_id)
WHERE (wp_terms.term_id IS NULL)

DELETE wp_term_taxonomy FROM wp_term_taxonomy
LEFT JOIN wp_terms ON (wp_term_taxonomy.term_id = wp_terms.term_id)
WHERE (wp_terms.term_id IS NULL)
```

### wp_term_relationships -> wp_term_taxonomy

```sql
SELECT * FROM wp_term_relationships
LEFT JOIN wp_term_taxonomy
	ON (wp_term_relationships.term_taxonomy_id = wp_term_taxonomy.term_taxonomy_id)
WHERE (wp_term_taxonomy.term_taxonomy_id IS NULL)

DELETE wp_term_relationships FROM wp_term_relationships
LEFT JOIN wp_term_taxonomy
	ON (wp_term_relationships.term_taxonomy_id = wp_term_taxonomy.term_taxonomy_id)
WHERE (wp_term_taxonomy.term_taxonomy_id IS NULL)
```

### wp_usermeta -> wp_users

```sql
SELECT * FROM wp_usermeta
LEFT JOIN wp_users ON (wp_usermeta.user_id = wp_users.ID)
WHERE (wp_users.ID IS NULL)

DELETE wp_usermeta FROM wp_usermeta
LEFT JOIN wp_users ON (wp_usermeta.user_id = wp_users.ID)
WHERE (wp_users.ID IS NULL)
```

### wp_posts -> wp_users
Delete posts where the user was deleted. Use with caution, old posts still may be used.

```sql
SELECT * FROM wp_posts
LEFT JOIN wp_users ON (wp_posts.post_author = wp_users.ID)
WHERE (wp_users.ID IS NULL)

DELETE wp_posts FROM wp_posts
LEFT JOIN wp_users ON (wp_posts.post_author = wp_users.ID)
WHERE (wp_users.ID IS NULL)
```

## Other

### wp_postmeta dupes
Checking for dupe `_wp_attached_file` / `_wp_attachment_metadata` keys (should only ever be one each per attachment post type).

```sql
SELECT post_id,meta_key,meta_value
FROM wp_postmeta
WHERE (meta_key IN('_wp_attached_file','_wp_attachment_metadata'))
GROUP BY post_id,meta_key
HAVING (COUNT(post_id) > 1)
```

### wp_postmeta dupes #2
Where an identical `meta_key` exists for the same post more than once.

```sql
SELECT *,COUNT(*) AS keycount
FROM wp_postmeta
GROUP BY post_id,meta_key
HAVING (COUNT(*) > 1)

DELETE FROM wp_postmeta
WHERE (meta_id IN (
	SELECT * FROM (
		SELECT meta_id
		FROM wp_postmeta tmp
		GROUP BY post_id,meta_key
		HAVING (COUNT(*) > 1)
	) AS tmp
))
```

### wp_postmeta missing
Checking for missing `_wp_attached_file` / `_wp_attachment_metadata` keys on `wp_posts.post_type = 'attachment'` rows.

```sql
SELECT * FROM wp_posts
LEFT JOIN wp_postmeta ON (
	(wp_posts.ID = wp_postmeta.post_id) AND
	(wp_postmeta.meta_key = '_wp_attached_file')
)
WHERE (wp_posts.post_type = 'attachment') AND (wp_postmeta.meta_id IS NULL)

SELECT * FROM wp_posts
LEFT JOIN wp_postmeta ON (
	(wp_posts.ID = wp_postmeta.post_id) AND
	(wp_postmeta.meta_key = '_wp_attachment_metadata')
)
WHERE (wp_posts.post_type = 'attachment') AND (wp_postmeta.meta_id IS NULL)
```

### wp_postmeta '_edit_lock' and  '_edit_last' rows
Rows created against a post when edited by a WordPress admin user. They can be [safely removed](https://wordpress.org/support/topic/can-i-remove-_edit_lock-_edit_last-from-wp_postmeta).

```sql
SELECT * FROM wp_postmeta
WHERE meta_key IN ('_edit_lock','_edit_last')

DELETE FROM wp_postmeta
WHERE meta_key IN ('_edit_lock','_edit_last')
```

### wp_options '_transient_' rows
A transient value is one stored by WordPress and/or a plugin generated from a complex query - basically a cache. More information can be found in this [answer on Stack Overflow](http://stackoverflow.com/a/11995022).

```sql
SELECT * FROM wp_options
WHERE option_name LIKE '%\_transient\_%'

DELETE FROM wp_options
WHERE option_name LIKE '%\_transient\_%'
```

### wp_posts revisions
Every save of a WordPress post will create a new revision (and related wp_postmeta rows). To clear out all revisions older than 15 days:

```sql
SELECT * FROM wp_posts
WHERE
	(post_type = 'revision') AND
	(post_modified_gmt < DATE_SUB(NOW(),INTERVAL 15 DAY))
ORDER BY post_modified_gmt DESC

DELETE FROM wp_posts
WHERE
	(post_type = 'revision') AND
	(post_modified_gmt < DATE_SUB(NOW(),INTERVAL 15 DAY))
```

You may need to run the [wp_postmeta -> wp_posts](#wp_postmeta---wp_posts) orphans query after cleaning up revisions.

## Orphaned Term Relationships
Step 1: Delete Term Relationships
I had already deleted all posts and postmeta, but the links between those post IDs and my taxonomies still existed.  The first step was to clean this up.
```
DELETE FROM wp_term_relationships
WHERE object_id NOT IN (SELECT ID FROM wp_posts);
```
Step 2: Update the Term Counts
The next step is to update the term counts for each taxonomy.  In deleting the hundreds of posts that I did, many of my terms would now be used 0 times, but others would still have at least one post in them.  This statement will clean all of that up.
```
UPDATE wp_term_taxonomy tt
SET count = (SELECT count(p.ID)
FROM wp_term_relationships tr
LEFT JOIN wp_posts p ON p.ID = tr.object_id
WHERE tr.term_taxonomy_id = tt.term_taxonomy_id);
```
Step 3: Delete Unused Terms
Finally, I ran the following statement to delete all terms that were no longer in use ( they have a count of 0 )
```
DELETE wt
FROM wp_terms a
INNER JOIN wp_term_taxonomy b ON a.term_id = b.term_id
WHERE b.count = 0;
```
