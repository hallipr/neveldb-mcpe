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
...