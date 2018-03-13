# neveldb-mcpe
A .net standard (hopefully) library for reading Mojang's zlib flavored LevelDB files.

## Getting my feet wet
I'll be taking some time to document LevelDB's read process before I start writing the library.  I'm working from Mojang's fork here: [Mojang/leveldb-mcpe](https://github.com/Mojang/leveldb-mcpe).  I'm not a c++ developer, and all of the source is in c++, so this will be fun.

### database.Get(key)
It looks like the magic starts in [db/db_impl.cc](https://github.com/Mojang/leveldb-mcpe/blob/master/db/db_impl.cc) at the definition of DB:Get()

    DB::Get(const ReadOptions& options, const Slice& key, std::string* value)

    ....

    Version current = versions_->current()
    current->Get(options, lkey, value, &stats);

In the constructor, we see versions_ = new VersionSet(dbname_, &options_, table_cache_, &internal_comparator_);

So we get the current version from our version set, then we ask that version to get the key.

### version.Get(key)
From [db/version_set.cc](https://github.com/Mojang/leveldb-mcpe/blob/master/db/version_set.cc):

    Status Version::Get(const ReadOptions& options, const LookupKey& k, std::string* value, GetStats* stats) {

It basically breaks down like this

    foreach(level in version) {
        File[] files;
        if currentLevel == 0 {
            build list of files, sorted newest to oldest, that may contain the key.
            files = all files that may contain the key
        } else {
            binary search to find earliest index whose largest key >= ikey.
            if index is >= files in the level, the key is > than all keys in this level
                files = NULL;
            } else {
                // Get the file that would contain the key if it's in this level
                tmp2 = files[index];
                // before scanning the file, make sure the key isn't smaller than all keys in this file
                if (ucmp->Compare(user_key, tmp2->smallest.user_key()) < 0) {
                    // All of "tmp2" is past any data for user_key
                    files = NULL;
                } else {
                    //Files to scan = this file
                    files = tmp2;
                }
            }
        }
        
        Scan "files" 
        foreach(file in files)
        {
            // looks like we need to ask the cache to pull the value from the file.
            // I guess this is just some preemptive caching.
            status = vset_.table_cache_.Get(options, file.number, file.size, ikey, &saver, SaveValue);
            
            if (state.ok())  // maybe a safety check?? 
            {
                switch(status)
                    case NotFound
                        continue;   // not this file, so keep looking
                    case Deleted
                        status = NotFound  // stop looking, it no longer exists
                    case Corrupt
                        Add More Detailed to status message
            }
            
            return status;   // Found, NotFound(Deleted), Corrupt, some weird Not OK status
        }
    }
    
    return NotFound
}

So.... what does asking a table cache to get a key look like?

### tableCache.Get(fileNumber, fileSize, key)

``` cpp
    Status TableCache::Get(const ReadOptions& options,
                        uint64_t file_number,
                        uint64_t file_size,
                        const Slice& k,
                        void* arg,
                        void (*saver)(void*, const Slice&, const Slice&)) {
    Cache::Handle* handle = NULL;
    Status s = FindTable(file_number, file_size, &handle);
    if (s.ok()) {
        Table* t = reinterpret_cast<TableAndFile*>(cache_->Value(handle))->table;
        s = t->InternalGet(options, k, arg, saver);
        cache_->Release(handle);
    }
    return s;
    }
```

We find the table, then if all is good, we ask the table to get the value for the key.
I think the ```cache_->Value(handle)``` does the caching. If so, we cache a table at a time, not a key/value pair.  

Nope, the caching happens here :)

``` cpp
Status TableCache::FindTable(uint64_t file_number, uint64_t file_size, Cache::Handle** handle) {
  lookup key in cache

  if (cache miss) {
    build file name
    open the file

    if (!s.ok()) {
      try again with an "old" file name...  maybe old=legacy?
    }
    
    if (s.ok()) {
      Table::Open(*options_, file, file_size, &table);
    }

    if (s.ok()) {
      insert the table and file into cache under the key
    }

    return s
  }
```

Once we have a table, we call table->InternalGet:

``` cpp
Status Table::InternalGet(const ReadOptions& options, const Slice& k, void* arg, void (*saver)(void*, const Slice&, const Slice&)) {
  get an index iterator and seek the key
  if (iiter->Valid()) {
    Slice handle_value = iiter->value();
    FilterBlockReader* filter = rep_->filter;
    BlockHandle handle;
    // see if the filter thinks we can ignore the handle value
    if (filter != NULL && handle.DecodeFrom(&handle_value).ok() &&
        !filter->KeyMayMatch(handle.offset(), k)) {
      // Not found
    } else {
      // we found a block that may contain the key, seek the block
      Iterator* block_iter = BlockReader(this, options, iiter->value());
      block_iter->Seek(k);
      if (block_iter->Valid()) {
        // looks like we output the value using a delegate "saver"
        // saver(arg, key, value)        
        (*saver)(arg, block_iter->key(), block_iter->value());
      }
      s = block_iter->status();
      delete block_iter;
    }
  }
  if (s.ok()) {
    s = iiter->status();
  }
  delete iiter;
  return s;
}
```


So, I completely ignored the "saver" delegate being passed down from Version and the state object arg beside it.   It points back to SaveValue in version_set.cc:

``` cpp
static void SaveValue(void* arg, const Slice& ikey, const Slice& v) {
  Saver* s = reinterpret_cast<Saver*>(arg);
  ParsedInternalKey parsed_key;
  if (!ParseInternalKey(ikey, &parsed_key)) {
    s->state = kCorrupt;
  } else {
    if (s->ucmp->Compare(parsed_key.user_key, s->user_key) == 0) {
      s->state = (parsed_key.type == kTypeValue) ? kFound : kDeleted;
      if (s->state == kFound && s->value) {
        s->value->assign(v.data(), v.size());
      }
    }
  }
}
``` 

Looks like we do a user -> internal key parse check, followed by a key comparison.  We make sure the key isn't a deletion placeholder before assigning the fetched data and size to the status object's value.

## So, top-down

``` c#
db.Get(key) {
    versionSet.Current.Get(key) {
        files = GetFilesToCheck
        foreach(file in files) {
            tableCache.Get(file, key) {
                var table = GetTableWithCache(file);
                table.Get(key) {
                    var block = index.get(key);
                    if(block && filter(block)) {
                        var kvPair = block.get(key)
                        return kvPair
                    }
                }
            }

            if(found) break;
        }
    }
}

