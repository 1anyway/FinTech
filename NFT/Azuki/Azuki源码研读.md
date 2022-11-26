# Azuki源码研读
## What is Azuki
Azuki是在OpenSea上发售的一系列日漫风格的PFP nfts。Azuki致力于将自身打造为由社区建造、拥有的最大的去中心化品牌之一。 
The features of Azuki
Azuki的代码主要突出了两个特点，一个是实现了ERC721A，批量铸造省去了ERC721只能一次一次mint的gas费，Azuki是最先创造ERC721A这一机制的。另一个则是荷兰拍卖，荷兰拍卖这种模式基本上也是Azuki带起来的。与通常意义上所理解的卖方设置最低价由出价高者拍得的拍卖不同，荷兰拍卖是卖方设置最高价，买方在拍卖是按照规则逐一降价。
## How it works
Azuki主要的功能就是通过两个合约来实现的，就是ERC721A以及Azuki合约。
## ERC721A
为了便于后面的理解，先贴出数据，下方是Azuki ERC721A合约的基本数据。
```
contract ERC721A is Context,ERC165,IERC721,IERC721Metadata,IERC721Enumerable {
  using Address for address;
  using Strings for uint256;

  struct TokenOwnership {
    address addr;
    uint64 startTimestamp;
  }

  struct AddressData {
    uint128 balance;
    uint128 numberMinted;
  }

  uint256 private currentIndex = 0;

  uint256 internal immutable collectionSize;
  
  //一次性可以铸造的nft的最大数量
  uint256 internal immutable maxBatchSize;

  // Token name
  string private _name;

  // Token symbol
  string private _symbol;

  // Mapping from token ID to ownership details
  // 当一个tokenId对应的TokenOwnership为空时，并不意味着该tokenId不存在，因为在铸造时只记录
  //每一次铸造的第一个nft的tokenId
  mapping(uint256 => TokenOwnership) private _ownerships;

  // Mapping owner address to address data
  mapping(address => AddressData) private _addressData;

  // Mapping from token ID to approved address
  mapping(uint256 => address) private _tokenApprovals;

  // Mapping from owner to operator approvals
  mapping(address => mapping(address => bool)) private _operatorApprovals;
```

在这里Azuki实现了它的重要功能，批量铸造。在铸造的时候，它并不记录所有铸造的tokenId的拥有者信息。任何一次批量铸造，它只记录该次铸造的第一个tokenId的拥有者信息。这样大大节省了gas费。
```
function _safeMint(
    address to,
    uint256 quantity,
    bytes memory _data
  ) internal {
    uint256 startTokenId = currentIndex;
    require(to != address(0), "ERC721A: mint to the zero address");
    // We know if the first token in the batch doesn't exist, the other ones don't as well, because of serial ordering.
    require(!_exists(startTokenId), "ERC721A: token already minted");
    require(quantity <= maxBatchSize, "ERC721A: quantity to mint too high");

    _beforeTokenTransfers(address(0), to, startTokenId, quantity);

    AddressData memory addressData = _addressData[to];
    _addressData[to] = AddressData(
      addressData.balance + uint128(quantity),
      addressData.numberMinted + uint128(quantity)
    );
    //只记录每次铸造的第一个tokenId的拥有关系.
    _ownerships[startTokenId] = TokenOwnership(to, uint64(block.timestamp));

    uint256 updatedIndex = startTokenId;

    for (uint256 i = 0; i < quantity; i++) {
      emit Transfer(address(0), to, updatedIndex);
      require(
        _checkOnERC721Received(address(0), to, updatedIndex, _data),
        "ERC721A: transfer to non ERC721Receiver implementer"
      );
      updatedIndex++;
    }

    currentIndex = updatedIndex;
    _afterTokenTransfers(address(0), to, startTokenId, quantity);
  }
```
通过下面的函数，通过实现往前追溯每一次mint的第一个tokenId的拥有者信息，可以返回每次铸造的每一个nft的tokenId的拥有者信息.
```
uint256 public nextOwnerToExplicitlySet = 0;
function _setOwnersExplicit(uint256 quantity) internal {
    uint256 oldNextOwnerToSet = nextOwnerToExplicitlySet;
    require(quantity > 0, "quantity must be nonzero");
    uint256 endIndex = oldNextOwnerToSet + quantity - 1;
    if (endIndex > collectionSize - 1) {
      endIndex = collectionSize - 1;
    }
    // We know if the last one in the group exists, all in the group exist, due to serial ordering.
    require(_exists(endIndex), "not enough minted yet for this cleanup");
    for (uint256 i = oldNextOwnerToSet; i <= endIndex; i++) {
      if (_ownerships[i].addr == address(0)) {
        TokenOwnership memory ownership = ownershipOf(i);
        _ownerships[i] = TokenOwnership(
          ownership.addr,
          ownership.startTimestamp
        );
      }
    }
    nextOwnerToExplicitlySet = endIndex + 1;
  }
  
  function ownershipOf(uint256 tokenId)
    internal
    view
    returns (TokenOwnership memory)
  {
    require(_exists(tokenId), "ERC721A: owner query for nonexistent token");

    uint256 lowestTokenToCheck;
    if (tokenId >= maxBatchSize) {
      lowestTokenToCheck = tokenId - maxBatchSize + 1;
    }

    for (uint256 curr = tokenId; curr >= lowestTokenToCheck; curr--) {
      TokenOwnership memory ownership = _ownerships[curr];
      if (ownership.addr != address(0)) {
        return ownership;
      }
    }
```

## Azuki
下面 是Azuki合约的基本数据
```
contract Azuki is Ownable, ERC721A, ReentrancyGuard {
  uint256 public immutable maxPerAddressDuringMint;
  uint256 public immutable amountForDevs;
  uint256 public immutable amountForAuctionAndDev;

//nft的发行分三种模式，一是指定价格公开售卖，二是拍卖的形式，三是白名单
  struct SaleConfig {
    uint32 auctionSaleStartTime;
    uint32 publicSaleStartTime;
    uint64 mintlistPrice;
    uint64 publicPrice;
    uint32 publicSaleKey;
  }

  SaleConfig public saleConfig;
  
  //存储白名单地址主要是两种方法，一是下面的Azuki项目方使用的比较粗糙的方法，每一次导入
  //白名单都将消耗大量的gas
  //更聪明的办法是构造白名单默克尔树.
  mapping(address => uint256) public allowlist;

  constructor(
    uint256 maxBatchSize_,
    uint256 collectionSize_,
    uint256 amountForAuctionAndDev_,
    uint256 amountForDevs_
  ) ERC721A("Azuki", "AZUKI", maxBatchSize_, collectionSize_) {
    maxPerAddressDuringMint = maxBatchSize_;
    amountForAuctionAndDev = amountForAuctionAndDev_;
    amountForDevs = amountForDevs_;
    require(
      amountForAuctionAndDev_ <= collectionSize_,
      "larger collection size needed"
    );
  }
  
 //Azuki项目方在这里使用了防科学家机制，通常在nft发售时，一些“科学家”会通过脚本的方式抢购.
 //一般的项目方是不会阻止这种行为的，他们正愁买不出去呢，反正有人付费把nft(JPG）卖出去就行.
 //但是Azuki项目对自己还是比较有信心的, 设置了检验脚本的程序，即只有EOA才能mint.
 //不过下面的方式，多签钱包拥有nft这种合理场景是没法考虑的.
  modifier callerIsUser() {
    //下面的方式，为true时，该账户一定是EOA.
    //另一种方式，通过Openzeppelin的Address.sol中的isContract()判断.
    //isContract()为true时一定不是EOA，但为false时，情况就有很多种了.
    require(tx.origin == msg.sender, "The caller is another contract");
    _;
  }
```
下面是荷兰拍卖.
```
  //拍卖最初设置最高价
  uint256 public constant AUCTION_START_PRICE = 1 ether;
  //拍卖结束地板价(最低价)
  uint256 public constant AUCTION_END_PRICE = 0.15 ether;
  //拍卖持续时间
  uint256 public constant AUCTION_PRICE_CURVE_LENGTH = 340 minutes;
  //每隔20分钟降一次价
  uint256 public constant AUCTION_DROP_INTERVAL = 20 minutes;
  //计算每次降价多少
  uint256 public constant AUCTION_DROP_PER_STEP =
    (AUCTION_START_PRICE - AUCTION_END_PRICE) /
      (AUCTION_PRICE_CURVE_LENGTH / AUCTION_DROP_INTERVAL);

  function getAuctionPrice(uint256 _saleStartTime)
    public
    view
    returns (uint256)
  {
    if (block.timestamp < _saleStartTime) {
      return AUCTION_START_PRICE;
    }
    if (block.timestamp - _saleStartTime >= AUCTION_PRICE_CURVE_LENGTH) {
      return AUCTION_END_PRICE;
    } else {
      uint256 steps = (block.timestamp - _saleStartTime) /
        AUCTION_DROP_INTERVAL;
      return AUCTION_START_PRICE - (steps * AUCTION_DROP_PER_STEP);
    }
  }
```