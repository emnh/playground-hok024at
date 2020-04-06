# Fast 15x15 bit grid BFS: breadth first search

We represent a 15x15 grid as 4 64 bit unsigned integers.
The last column, 16th, and row, also 16th, should always be empty.

Each presence of anything on the map is represented as a bit set to 1 and absence of anything is the bit set to 0.
To get the neighbours of current set cells we shift bits left, right, up or down
and then to get both we take the union with current set cells (bitwise or).
To restrict from going into unavailable cells we do intersection (bitwise and) with inversion (bitwise not) of unavailable cells.
As we said the last row and column are unavailable, and also "islands" (in this case for Ocean of Code) are unavailable.

The optimization idea is to split the board into white and black cells by odd and even indices, like a chess board.
In that way black cells only have white neighbours and white cells only have black neighbours.

So the first 64 bit int (index 0) contains cells where x is even and y is even, i.e white cells.
The second 64 bit int (index 1) contains cells where x is odd and y is even, i.e. black cells.
The third 64 bit int (index 2) contains cells where x is even and y is odd, i.e. black cells.
The fourth 64 bit int (index 3) contains cells where x is odd and y is odd, i.e. white cells.

For certain 2.4 GHz servers on CodinGame you can get between 1.5 and 2 million BFSes per seconds.
For the 2.2 GHz server it's around 1.3-1.4 million.
On tech.io it is much slower for some reason, a few hundred k is all you get it seems.

You no longer should have to worry about whether starting cell is black or white.
Code has been modified at expense of a tiny bit of performance.
If you know whether your starting cell is black or white you can optimize the code
to make the first expansion from black to white or white to black accordingly.


```C++ runnable
#pragma GCC optimize("Ofast","unroll-loops","omit-frame-pointer","inline")
#include <iostream>
#include <string>
#include <chrono>
#ifdef __GNUC__
#include <cfenv>
#endif

using namespace std;

using time_interval_t = std::chrono::microseconds;
using myClock = std::chrono::high_resolution_clock;

const int W = 15;
const int H = 15;
const int WP = W + 1;
const int BitsPerRow = WP / 2 + 0;
const int B = 4;

struct ExpandReturn
{
    uint64_t out0or2;
    uint64_t out3or1;
    bool done;
};

class BitMapGrid
{
public:
    uint64_t bits[B] = { 0, 0, 0, 0 };

    BitMapGrid Invert()
    {
        BitMapGrid ret;
	    for (int i = 0; i < B; i++)
	    {
            ret.bits[i] = ~bits[i];
	    }
        return ret;
    }

    static void GetIndex(
        unsigned int x,
        unsigned int y,
        unsigned int& bitIndex,
        unsigned int& intIndex)
    {
        // x even, y even: 0: white
        // x odd, y even: 1: black
        // x even, y odd: 2: black
        // x odd, y odd: 3: white
        // 01
        // 23

        // Every 8th bit supposed to be unused
        // (except for "available" grid),
        // to guard shifts between rows and columns.
        // So map last row onto those.
        /*
        if (y == 15) {
          y = min(14u, x + 2);
          x = -2;
          //intIndex = (x % 2) + ((y % 2) * 2);
          //bitIndex = ((y / 2) * BitsPerRow) + (x / 2);
          //return;
        }*/

        intIndex = (x % 2) + ((y % 2) * 2);
        bitIndex = ((y / 2) * BitsPerRow) + (x / 2);
        if (bitIndex >= 64) {
          int k = 0;
          cerr << "INVALID BITINDEX FOR XY: " << x << "," << y << endl;
        }
    }

    BitMapGrid Intersection(const BitMapGrid& other) const {
      BitMapGrid ret;
      for (int i = 0; i < B; i++) {
        ret.bits[i] = this->bits[i] & other.bits[i];
      }
      return ret;
    }

    BitMapGrid GetAvailable() {
      BitMapGrid full;
      // Make sure all used bits are set and unused bits are unset
      for (int y = 0; y < H; y++) {
          for (unsigned int x = 0; x < W; x++) {
            full.Set(x, y);
          }
      }
      return Intersection(full).Invert();
    }

    static uint64_t ShiftBytesLeft(const uint64_t& a)
    {
        return (a & 0x7F7F7F7F7F7F7F7Full) << 1;
        //return a << 1;
    }

    static uint64_t ShiftBytesRight(const uint64_t& a)
    {
        return (a & 0xFEFEFEFEFEFEFEFEull) >> 1;
        //return a >> 1;
    }

    static ExpandReturn ExpandPlusCommon(
        const uint64_t grid0,
        const uint64_t grid1,
        const uint64_t grid2,
        const uint64_t grid3,
        const uint64_t available0,
        const uint64_t available3,
        const uint64_t grid0Or,
        const uint64_t grid3Or
        )
    {
        const uint64_t grid0Old = grid0;
        const uint64_t grid3Old = grid3;

        const uint64_t common = grid1 | grid2;
        //const uint64_t a = (grid0 | common | grid0Or | grid2 << BitsPerRow) & available0;
        //const uint64_t b = (grid3 | common | grid3Or | grid1 >> BitsPerRow) & available3;
        const uint64_t a = (grid0 | common | grid0Or) & available0;
        const uint64_t b = (grid3 | common | grid3Or) & available3;

        return { a, b, a == grid0Old && b == grid3Old };
    };

    static BitMapGrid ExpandPlusCombine(
        const BitMapGrid& gridInput, const BitMapGrid& available, int& i) {

        uint64_t gridBits0 = gridInput.bits[0];
        uint64_t gridBits1 = gridInput.bits[1];
        uint64_t gridBits2 = gridInput.bits[2];
        uint64_t gridBits3 = gridInput.bits[3];

        const auto& PP = [&](string header) {
          BitMapGrid grid;
          grid.bits[0] = gridBits0;
          grid.bits[1] = gridBits1;
          grid.bits[2] = gridBits2;
          grid.bits[3] = gridBits3;
          cerr << header << endl;
          grid.Print();
        };

        while (true)
        {
            {
                // Returning 2 and 1
                // NOTE: Can optimize to avoid the first call to ExpandPlusCommon if starting cell is black.
                // As then the first expansion is from black to white, not white to black as assumed here.
                const ExpandReturn ret =
                    ExpandPlusCommon(
                        gridBits2, gridBits3, gridBits0, gridBits1,
                        available.bits[2], available.bits[1],
                        ShiftBytesLeft(gridBits3) | (gridBits0 >> BitsPerRow),
                        ShiftBytesRight(gridBits0) | (gridBits3 << BitsPerRow));
                gridBits1 = ret.out3or1;
                gridBits2 = ret.out0or2;
                //PP("RET21");
                if (i > 0 && ret.done)
                {
                    break;
                }
                if (!ret.done) i++;

                // Returning 0 and 3
                const ExpandReturn ret2 =
                    ExpandPlusCommon(
                        gridBits0, ret.out3or1, ret.out0or2, gridBits3,
                        available.bits[0], available.bits[3],
                        ShiftBytesLeft(ret.out3or1) | (ret.out0or2 << BitsPerRow),
                        ShiftBytesRight(ret.out0or2) | (ret.out3or1 >> BitsPerRow));
                gridBits0 = ret2.out0or2;
                gridBits3 = ret2.out3or1;
                //PP("RET30");
                if (ret2.done)
                {
                    break;
                };
                i++;
            }

            // Mostly just duplicate of above: Manual loop unrolling.
            {
                const ExpandReturn ret =
                    ExpandPlusCommon(
                        gridBits2, gridBits3, gridBits0, gridBits1,
                        available.bits[2], available.bits[1],
                        ShiftBytesLeft(gridBits3) | (gridBits0 >> BitsPerRow),
                        ShiftBytesRight(gridBits0) | (gridBits3 << BitsPerRow));
                gridBits1 = ret.out3or1;
                gridBits2 = ret.out0or2;
                //PP("RET21");
                if (ret.done)
                {
                    break;
                };
                i++;


                // Returning 0 and 3
                const ExpandReturn ret2 =
                    ExpandPlusCommon(
                        gridBits0, ret.out3or1, ret.out0or2, gridBits3,
                        available.bits[0], available.bits[3],
                        ShiftBytesLeft(ret.out3or1) | (ret.out0or2 << BitsPerRow),
                        ShiftBytesRight(ret.out0or2) | (ret.out3or1 >> BitsPerRow));
                gridBits0 = ret2.out0or2;
                gridBits3 = ret2.out3or1;
                //PP("RET30");
                if (ret2.done)
                {
                    break;
                };
                i++;
            }
        }

        BitMapGrid grid;
        grid.bits[0] = gridBits0;
        grid.bits[1] = gridBits1;
        grid.bits[2] = gridBits2;
        grid.bits[3] = gridBits3;
        return grid;
    }

    bool Get(unsigned int x, unsigned int y) const
    {
        unsigned int bitIndex;
        unsigned int intIndex;
        GetIndex(x, y, bitIndex, intIndex);
        return (bits[intIndex] >> bitIndex) & 1ULL;
    }

    void Set(unsigned int x, unsigned int y)
    {
        unsigned int bitIndex;
        unsigned int intIndex;
        GetIndex(x, y, bitIndex, intIndex);
        bits[intIndex] |= 1ULL << bitIndex;
    }

    void Clear(unsigned int x, unsigned int y)
    {
        unsigned int bitIndex;
        unsigned int intIndex;
        GetIndex(x, y, bitIndex, intIndex);
        bits[intIndex] &= ~(1ULL << bitIndex);
    }

    static string pad(int x) {
      return x < 10 ? "0" : "";
    }

    void Print() const
    {
        unsigned char out[(W + 1) * H + 1];
        for (unsigned int y = 0; y < H; y++)
        {
            for (unsigned int x = 0; x < W; x++)
            {
                unsigned int bitIndex;
                unsigned int intIndex;
                GetIndex(x, y, bitIndex, intIndex);
                /*
                cerr << pad(x) << x << "," << pad(y) << y << ": " <<
                  pad(intIndex) << intIndex << "," <<
                  pad(bitIndex) << bitIndex << " ";
                  */
                out[x + y * (W + 1)] =
                    Get(x, y) ? '0' + intIndex : '.';
            }
            //cerr << endl;
            out[(y + 1) * (W + 1) - 1] = '\n';
        }
        out[(W + 1) * H] = 0;
        cerr << out << endl;
    }
};

void test(const BitMapGrid& available, const int timeout)
{
    const auto start = myClock::now();
    int maxIterations = 0;

    const uint64_t count = 120000;
    BitMapGrid toPrint;
    for (unsigned int k = 0; k < count; k++) {
        BitMapGrid g;
        g.Set(7, 7);

        int i = 0;
        g = BitMapGrid::ExpandPlusCombine(g, available, i);
        maxIterations = i;
        toPrint = g;
    }

    const auto elapsed2 =
        std::chrono::duration_cast<time_interval_t>(myClock::now() - start);
    cerr
		<< "COUNT: " << count << " "
        << "COUNT PER 50ms: " << count * 50000 / elapsed2.count() << " / "
        << "ELAPSED: " << elapsed2.count() << " microseconds" << endl;

    cerr << "ITERATIONS PER BFS (BREADTH): " << maxIterations << endl;
    toPrint.Print();
}

void game()
{
    int width;
    int height;
    int myId;
    cin >> width >> height >> myId; cin.ignore();
    BitMapGrid islands;
    for (int y = 0; y < height; y++) {
        string line;
        getline(cin, line);
        for (unsigned int x = 0; x < line.length(); x++)
        {
            if (line[x] == 'x')
            {
                islands.Set(x, y);
            }
        }
    }
    const BitMapGrid available = islands.GetAvailable();

    test(available, 950000);

    cout << "7 7" << endl;

    // game loop
    while (!cin.eof()) {
        int x;
        int y;
        int myLife;
        int oppLife;
        int torpedoCooldown;
        int sonarCooldown;
        int silenceCooldown;
        int mineCooldown;
        cin >> x >> y >> myLife >> oppLife >> torpedoCooldown >> sonarCooldown >> silenceCooldown >> mineCooldown; cin.ignore();

        test(available, 45000);

        string sonarResult;
        cin >> sonarResult; cin.ignore();
        string opponentOrders;
        getline(cin, opponentOrders);

        cout << "MOVE N TORPEDO" << endl;
    }
}

int main()
{
    system("cat /proc/cpuinfo | grep \"model name\" | head -1 >&2");
    system("cat /proc/cpuinfo | grep \"cpu MHz\" | head -1 >&2");

    ios::sync_with_stdio(false);
#ifdef __GNUC__
    feenableexcept(FE_DIVBYZERO | FE_INVALID | FE_OVERFLOW);
#endif

    BitMapGrid islands;
    for (int i = 1; i < 15; i++) {
     // islands.Set(2, i);
    }
    // 2x2
    /*
islands.Set(1, 1);
islands.Set(1, 2);
islands.Set(2, 1);
islands.Set(2, 2);

// close the corner
islands.Set(2, 0);
islands.Set(0, 2);
islands.Print();
*/

    cerr << endl;

    const BitMapGrid available = islands.GetAvailable();
    test(available, 45000);

    //game();
    return 0;
}
```
