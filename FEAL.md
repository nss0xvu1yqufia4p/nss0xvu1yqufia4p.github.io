# Differential Cryptanalysis of FEAL - implementation

Just a short C++ implementation of an attack on the FEAL-4 block cipher outlined here: [http://theamazingking.com/crypto-feal.php](http://theamazingking.com/crypto-feal.php)

```cpp
//This code requires c++11 compile with g++ -std=c++11
//Found A to be 0x006c1800

#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include <inttypes.h>
#include <limits.h>
#include <assert.h>
#include <vector>
#include <iostream>
#include <algorithm>
#include <utility>
#include <map>
#include <numeric>
#include <set>
#include <functional>

using namespace std;

//left rotation, adapted from 
//stackoverflow.com/questions/776508/best-practices-for-circular-shift-rotate-operations-in-c
static inline uint8_t rotl8_2 (uint8_t n) {
	const unsigned int mask = (CHAR_BIT*sizeof(n) - 1);  // assumes width is a power of 2.
  	unsigned int c = 2 & mask;
  	return (n<<c) | (n>>( (-c)&mask ));
}
//used for getting the left or right parts of different sized blocks
uint32_t left(uint64_t input) {
	return (uint32_t)(input>>32);
}
uint32_t right(uint64_t input) {
	return (uint32_t)input;
}
uint16_t left(uint32_t input) {
	return (uint16_t)(input>>16);
}
uint16_t right(uint32_t input) {
	return (uint16_t)input;
}
uint8_t left(uint16_t input) {
	return (uint8_t)(input>>8);
}
uint8_t right(uint16_t input) {
	return (uint8_t)input;
}
//concatenates 2 blocks
uint64_t concat(uint32_t left, uint32_t right) {
	return ((uint64_t)left<<32) | (uint64_t)right;
}
uint32_t concat(uint16_t left, uint16_t right) {
	return ((uint32_t)left<<16) | (uint64_t)right;
}
uint16_t concat(uint8_t left, uint8_t right) {
	return ((uint16_t)left<<8) | (uint16_t)right;
}
uint16_t mid16(uint32_t input) {
	return concat(right(left(input)), left(right(input)));
}
//define the functions used in the cipher
uint8_t G_0(uint8_t a, uint8_t b) {
	return rotl8_2(a+b); 
}
uint8_t G_1(uint8_t a, uint8_t b) {
	return rotl8_2(a+b+1); 
}
uint32_t F(uint32_t x) {
	uint8_t x_0 = left(left(x));
	uint8_t x_1 = right(left(x));
	uint8_t x_2 = left(right(x));
	uint8_t x_3 = right(right(x));
	uint8_t y_1 = G_1(x_0^x_1, x_2^x_3);
	uint8_t y_0 = G_0(x_0, y_1);
	uint8_t y_2 = G_0(y_1, x_2^x_3);
	uint8_t y_3 = G_1(y_2, x_3);
	return concat(concat(y_0, y_1), concat(y_2, y_3));
}
uint32_t M(uint32_t W) {
	uint8_t zero = 0;
	uint8_t b_0 = left(left(W));
	uint8_t b_1 = right(left(W));
	uint8_t b_2 = left(right(W));
	uint8_t b_3 = right(right(W));
	return concat(concat(zero, b_0^b_1), concat(b_2^b_3, zero));
}

typedef pair<uint64_t, uint64_t> TextPair;

int main() {
	vector<TextPair> C; //Vector of ciphertext pairs
	map<uint32_t, int> K; //Subkey 3 candidates

	C.push_back(TextPair(0xC5A28940BA4404D5, 0x46CA82A3C1A4EEB1));
	C.push_back(TextPair(0xAE2E85104CA3503A, 0x305BAE5E6A5E9AF0));
	C.push_back(TextPair(0x5ED24A0C6B88048F, 0x39D3D508A401BB0B));
	C.push_back(TextPair(0x2ABDDDE91DF1DE33, 0x0EAC11075168F358));

	uint32_t diff_0 = 0x80800000;
	uint32_t diff_1 = 0x02000000;
	
	for (auto & c: C) {
		printf("Starting Ciphertext pair 0x%" PRIx64 ", 0x%" PRIx64  "\n", c.first, c.second);
		uint32_t L_0 = left(c.first);
		uint32_t R_0 = right(c.first);
		uint32_t Y_0 = L_0^R_0;
		uint32_t L_1 = left(c.second);
		uint32_t R_1 = right(c.second);
		uint32_t Y_1 = L_1^R_1;
		uint32_t L_p = L_0^L_1;
		uint32_t Z_p = L_p^diff_1;
		vector<uint16_t> A; //Primary algo candidates
		for (uint32_t i=0; i<(0xFFFF+1); ++i) {
			uint16_t j = i;
			uint8_t zero = 0;
			uint8_t a_0 = left(j);
			uint8_t a_1 = right(j);
			uint32_t Q_0 = F(M(Y_0)^concat(concat(zero, a_0), concat(a_1, zero)));
			uint32_t Q_1 = F(M(Y_1)^concat(concat(zero, a_0), concat(a_1, zero)));
			if (mid16(Q_1^Q_0)==mid16(Z_p)) {
				A.push_back(j);
			}
		}
		printf("Found %d candidates for A=M(K3)\n", int(A.size()));
		int kcount = 0;
		for (auto & a:A) {
			uint8_t a_0 = left(a);
			uint8_t a_1 = right(a);
			for (uint32_t i=0; i<(0xFFFF+1); ++i) {
				uint16_t j = i;
				uint8_t c_0 = left(j);
				uint8_t c_1 = right(j);
				uint32_t D = concat(concat(c_0, a_0^c_0), concat(a_1^c_1, c_1));
				uint32_t Z_0_hat = F(Y_0^D);
				uint32_t Z_1_hat = F(Y_1^D);
				if ((Z_0_hat^Z_1_hat) == Z_p) {
					K[D]++;
					kcount++;
				}
			}
		}	
		printf("Found %d Candidate Keys for this ciphertext pair\n\n", kcount);
	}
	printf("Found %d candidate keys in total\n\n", int(K.size()));
	
	//The rest of the code is just manipulating stl containers to output stats
	vector<int> counts;
	std::transform(K.begin(), K.end(), std::back_inserter(counts),
		[](std::pair<const uint32_t, int>&p){return p.second;});
	sort(counts.begin(), counts.end());
	counts.erase(unique(counts.begin(), counts.end()), counts.end());
	map<int, int> countmap;
	for (auto & count:counts) {
		vector<int> temp;
		std::transform(K.begin(), K.end(), std::back_inserter(temp), 
			[count](std::pair<const uint32_t, int>&p){return (p.second==count)?1:0;});
		countmap[count]= std::accumulate(temp.begin(), temp.end(), 0);
	}
	for (auto & e:countmap) {
		cout << "There are " << e.second << " Candidate keys with a score of " << e.first <<endl;
	}
	vector<uint32_t> high_score_keys;
	cout << "\nHighest Score is: " << counts.back() << endl;
	cout << "High Scoreing Candidate Keys are:\n";
	set<uint32_t> A_vals;
	for (auto & e:K) {
		if (e.second==counts.back()) {
			A_vals.insert(M(e.first));
			printf("0x%" PRIx32 " Corresponding M(K3) is 0x%" PRIx32 "\n", e.first, M(e.first));
		}
	}
	assert(A_vals.size()==1);
	printf("\nCorrect Value for A=M(k3) is: 0x%" PRIx32 "\n", *A_vals.begin());		
}
```

