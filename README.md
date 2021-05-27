# cedar
C++ implementation of efficiently-updatable double-array trie


# About
Cedar implements an updatable double-array trie [1,2,3], which offers fast update/lookup for skewed queries in real-world data, e.g., counting words in text or mining conjunctive features in a classifier. Its update and lookup speed is comparable to (hard-to-serialize) hash-based containers (std::unordered_map) or modern cache-conscious tries [3,4] and even faster when the queries are short and skewed (see performance comparison).

The cedar is meant for those who still underestimate a double-array trie as a mutable container. The double-array tries are no longer slow to update. It's actually fast.

# Features
Fast update for skewed/ordered queries: Cedar can incrementally build a double-array trie from a given keyset in time that is less than the other double-array trie libraries. As a dynamic container, the ordered insertion is 3x faster than std::unordered_map, while the random insertion is comparable.
Fast lookup for skewed/ordered queries: Cedar, double-array trie, offers 2-30x faster skewed lookup than std::unordered_map, while the random lookup is comparable.
Serialization (load and store): Cedar can immediately de/serialize the resulting trie from/into disk as immutable/mutable dictionary, thanks to its simple, pointer-free data structure based on two (or three) one-dimensional arrays.
Small working space: Cedar, double-array trie, compactly stores a keyset that shares common prefixes, and it needs smaller working space (in building a trie) than other trie libraries that support serialization. If a static trie suffices and you mind the size of a resulting trie, I recommend darts-clone (same lookup speed, half size), or marisa-trie.
Native support for 4-byte record: Cedar stores up-to four-byte record in type specified by the first template parameter. The second and third template parameters specify two exceptional values that indicate specific lookup failures; no value or no path. The default exceptional values for no value / path are -1 / -2 for int record and NaN values for float record. You can of course associate a five- or more byte record with a key by preparing a value array and store its index (int) in the trie.
Sorted keys: The keys in a trie are alphabetically sorted as in std::map unless a user sets the third template parameter SORT_KEYS to false (slightly faster update).
Parameter to control space/time trade-off for binary keyset [2]: Setting a larger value to the fifth template parameter lets cedar thoroughly seek for empty elements in a trie to reduce its size. The default value usually gives a good trade-off in building a trie with low branching factor or fan-out. Currently, a binary key with '\0' inside is not supported.
Three trie implementations: a (normal) trie, a reduced trie [3] (compact size and faster look-up for short keys), a minimal-prefix trie (compact size for long keys). A reduced trie is enabled if you put #define USE_REDUCED_TRIE 1 before #include <cedar.h>, while a minimal-prefix trie is enabled if you #include <cedarpp.h> instead of cedar.h.
Rule of thumb: First consider to use a (versatile) minimal-prefix trie (#include <cedarpp.h>) when you wonder which trie to use for your datasets or benchmark experiments with various kinds of keys. Only when the keys are short, use a normal trie (#include <cedar.h>). Use the reduced trie (#include <cedar.h> preceded by #include USE_REDUCED_TRIE) cautiously since it has some limitations on the values as stated above. The normal trie and the reduced trie are not meant for long keys.
Simple, short, portable code: Ceder is implemented as a C++ header in ~600 lines, without depending on C++-specific headers such as standard template libraries.

# Requirements
OS: 32/64-bit UNIX-compatible OS (tested on Linux / Mac OS X)
Compiler: tested with GNU gcc (≥ 4.0) or clang (≥ 2.9)

# ToDo
Support compaction after erasing keys (shrink_to_fit()).
Support the zero-length key (empty string).
Support keys including '\0'.

# Usage
Cedar provides the following class template:

template <typename value_type,
          const int     NO_VALUE  = nan <value_type>::N1,
          const int     NO_PATH   = nan <value_type>::N2,
          const bool    ORDERED   = true,
          const int     MAX_TRIAL = 1,
          const size_t  NUM_TRACKING_NODES = 0>
class da;

This declares a double array trie with value_type as record type; NO_VALUE and NO_PATH are error codes returned by exactMatchSearch() or traverse() if search fails due to no path or no value, respectively. The keys in a trie is sorted if ORDERED is set to true, otherwise siblings are stored in an inserted order (slightly faster update). MAX_TRIAL is used to control space and time in online trie construction (useful when you want to build a trie from binary keys). If NUM_TRACKING_NODES is set to a positive integer value, the trie keeps node IDs stored in its public data member tracking_node to be valid, even if update() relocates the nodes to different locations.

The default NO_VALUE/NO_PATH values for int record are -1 and -2 (same as darts-variants), while the values for float record are 0x7f800001 and 0x7f800002 (taken from NaN values in IEEE 754 single-precision binary floating-point format); they can be referred to by CEDAR_NO_VALUE and CEDAR_NO_PATH, respectively.

NOTE: value_type must be in less than or equal to four (or precisely sizeof (int)) bytes. This does not mean you cannot associate a key with a value in five or more bytes (e.g., int64_t, double or user-defined struct). You can associate any record with a key by preparing a value array by yourself and store its index in (int) the trie (cedar::da <int>).

Cedar supports legacy trie APIs adopted in darts-clone, while providing new APIs for updating a trie.

value_type& update (const char* key, size_t len = 0, value_type val = value_type (0))
Insert key with length = len and value = val. If len is not given, std::strlen() is used to get the length of key. If key has been already present int the trie, val is added to the current value by using operator+=. When you want to override the value, omit val and write a value onto the reference to the value returned by the function.
int erase (const char* key, size_t len = 0, size_t from = 0)
Erase key (suffix) of length = len at from (root node in default) in the trie if exists. If len is not given, std::strlen() is used to get the length of key. erase() returns -1 if the trie does not include the given key. Currently, erase() does not try to refill the resulting empty elements with the tail elements for compaction.
template <typename T>
size_t commonPrefixPredict (const char* key, T* result, size_t result_len, size_t len = 0, size_t from = 0)
Predict suffixes following given key of length = len from a node at from, and stores at most result_len elements in result. result must be allocated with enough memory by a user. To recover keys, supply result in type cedar::result_triple_type (members are value, length, and id) and supply id and length to the following function, suffix(). The function returns the total number of suffixes (including those not stored in result).
template <typename T>
void dump (T* result, const size_t result_len)
Recover all the keys from the trie. Use suffix() to obtain actual key strings (this function works as commonPrefixPredict() from the root). To get all the results, result must be allocated with enough memory (result_len = num_keys()) by a user.

NOTE: The above two functions are implemented by the following two tree-traversal functions, begin() and next(), which enumerate the leaf nodes of a given tree by a pre-order walk; to predict one key by one, use these functions directly.
int begin (size_t& from, size_t& len)
Traverse a (sub)tree rooted by a node at from and return a value associated with the first (left-most) leaf node of the subtree. If the trie has no leaf node, it returns CEDAR_NO_PATH. Upon successful completion, from will point to the leaf node, while len will be the depth of the node. If you specify some internal node of a trie as from (in other words from != 0), remember to specify the depth of that node in the trie as len.
int next (size_t& from, size_t& len, const size_t root = 0)
Traverse a (sub)tree rooted by a node at root from a leaf node of depth len at from and return a value of the next (right) leaf node. If there is no leaf node at right-hand side of the subtree, it returns CEDAR_NO_PATH. Upon successful completion, from will point to the next leaf node, while len will be the depth of the node. This function is assumed to be called after calling begin() or next().
void suffix (char* key, const size_t len, size_t to)
Recover a (sub)string key of length = len in a trie that reaches node to. key must be allocated with enough memory by a user (to store a terminal character, len + 1 bytes are needed). Users may want to call some node-search function to obtain a valid node address and the (maximum) value of the length of the suffix.
void restore()
When you load an immutable double array, extra data needed to do predict(), dump() and update() are on-demand recovered when the function executed. This will incur some overhead (at the first execution). To avoid this, a user can explicitly run restore() just after loading the trie. This command is not defined when you configure --enable-fast-load since the configuration allows you to directly save/load a mutable double array.
There are three major updates to the legacy APIs.

build() accepts unsorted keys.
exactMatchSearch() returns CEDAR_NO_VALUE if search fails.
traverse() returns CEDAR_NO_VALUE if the key is present as prefix in the trie (but no value is associated), while it returns CEDAR_NO_PATH if the key is not present even as prefix.
Because update() can relocate the node id (from) obtained by traverse(), a user is advised to store the traversed node IDs in public data member tracking_node so that cedar can keep the IDs updated. The maximum number of elements in tracking_node should be specified as the sixth template parameter (0 is used as a sentinel).
Performance Comparison
Implementations
We compared cedar (2014-06-24) with the following in-memory and mutable containers that support sequential insertions.

Judy Array 1.0.5: Judy trie SL [11]
hat-trie: HAT-trie [12]
array-hash Array Hash: (cache-conscious) hash table [13]
hopscotch-map: Hopscotch hash [14]
sparsepp: sparse hash table
sparsehash 2.0.2: dense hash table
std::unordered_map <const char*, int> (gcc-7.1): hash table
These containers do not support serialization. We also compared cedar with the trie implementations that support serializations; the softwares other than cedar implement only static or immutable data structures that do not support update.

Darts 0.32: double-array trie
Darts-clone 0.32g: directed acyclic word graph
Darts-clone 0.32e5: Compacted double-array trie [15]
tx-trie*: LOUDS (Level-Order Unary Degree Sequence) trie [16]
ux-trie*: LOUDS double-trie
marisa-trie*: LOUDS nested patricia trie [17]
The succinct tries (*) do not store records; it needs extra space to store records (sizeof (int) * #keys bytes) and additional time to lookup it.

Unless otherwise versions are stated, codes in the latest snapshots of git or svn repository were used.

Settings (updating now...)
Having a set of keys and a set of query strings, we first build a container that maps a key to a unique value, and then look up the queries to the container. The codes used to benchmark containers (bench.cc and bench_static.cc) have been updated from the ones included in the latest cedar package. The slow-to-build or slow-to-lookup comparison-based containers and immutable double arrays are excluded from the benchmark (the previous results are still available from here). The entire keys (and queries) are loaded into memory to exclude I/O overheads before insertion (and look-up). All the hash-based containers use const char* in stead of expensive std::string to miminize the memory footprint and to exclude any overheads, and adopt CityHash64 from Google's cityhash in stead of default hash functions for a fair comparison. To compute the maximum memory footprint required in inserting the keys into the data structures, we subtracted the memory usage after loading keys or queries into memory from the maximum memory usage of the process monitored by run.cc (for mac) (or run_linux.cc).

Since we are interested in the lookup performance for practical situations where a small number of frequent keys are queried significantly more than a large number of rare ones (Zipfian distribution), the following datasets are used for experiments.

Text dataset from Dr. Askitis's website (see Datasets):
Key: distinct_1
Query: skew1_1
Binary dataset (conjunctive features (integers) extracted from pecco dataset and encoded by variable byte coding):
Key: keys3 (sorted): extracted and uniquified from pecco-YYYY-MM-DD/test/kernel_m3
Query: test3: extracted from pecco-YYYY-MM-DD/test/tl.dev
The statistics of keyset are summarized as follow (a terminal symbol included):

Linkage: http://www.tkl.iis.u-tokyo.ac.jp/~ynaga/cedar/#dl
