- Fast, secure, efficient backup program
- https://github.com/restic/restic
-
- 设计文档参考
	- https://restic.readthedocs.io/en/latest/100_references.html#design
-
- Repository Layout 设计
	- ```text
	  /tmp/restic-repo
	  ├── config
	  ├── data
	  │   ├── 21
	  │   │   └── 2159dd48f8a24f33c307b750592773f8b71ff8d11452132a7b2e2a6a01611be1
	  │   ├── 32
	  │   │   └── 32ea976bc30771cebad8285cd99120ac8786f9ffd42141d452458089985043a5
	  │   ├── 59
	  │   │   └── 59fe4bcde59bd6222eba87795e35a90d82cd2f138a27b6835032b7b58173a426
	  │   ├── 73
	  │   │   └── 73d04e6125cf3c28a299cc2f3cca3b78ceac396e4fcf9575e34536b26782413c
	  │   [...]
	  ├── index
	  │   ├── c38f5fb68307c6a3e3aa945d556e325dc38f5fb68307c6a3e3aa945d556e325d
	  │   └── ca171b1b7394d90d330b265d90f506f9984043b342525f019788f97e745c71fd
	  ├── keys
	  │   └── b02de829beeb3c01a63e6b25cbd421a98fef144f03b9a02e46eff9e2ca3f0bd7
	  ├── locks
	  ├── snapshots
	  │   └── 22a5af1bdc6e616f8a29579458c49627e01b32210d09adb288d1ecda7c5711ec
	  └── tmp
	  ```
- Pack Struct
	- ```text
	  EncryptedBlob1 || ... || EncryptedBlobN || EncryptedHeader || Header_Length
	  ```
	- Pack header
		- ```text
		  Type_Blob1 || Length(EncryptedBlob1) || Hash(Plaintext_Blob1) ||
		  [...]
		  Type_BlobN || Length(EncryptedBlobN) || Hash(Plaintext_Blobn) ||
		  ```
- Index
	- > When the local cached index is not accessible any more, the index files can be downloaded and used to reconstruct the index.
	- Format
		- ```JSON
		  {
		    "supersedes": [
		      "ed54ae36197f4745ebc4b54d10e0f623eaaaedd03013eb7ae90df881b7781452"
		    ],
		    "packs": [
		      {
		        "id": "73d04e6125cf3c28a299cc2f3cca3b78ceac396e4fcf9575e34536b26782413c",
		        "blobs": [
		          {
		            "id": "3ec79977ef0cf5de7b08cd12b874cd0f62bbaf7f07f3497a5b1bbcc8cb39b1ce",
		            "type": "data",
		            "offset": 0,
		            "length": 25
		          },{
		            "id": "9ccb846e60d90d4eb915848add7aa7ea1e4bbabfc60e573db9f7bfb2789afbae",
		            "type": "tree",
		            "offset": 38,
		            "length": 100
		          },
		          {
		            "id": "d3dc577b4ffd38cc4b32122cabf8655a0223ed22edfd93b353dc0c3f2b0fdf66",
		            "type": "data",
		            "offset": 150,
		            "length": 123
		          }
		        ]
		      }, [...]
		    ]
		  }
		  ```
- Snapshots
	- > All content within a restic repository is referenced according to its SHA-256 hash
	-