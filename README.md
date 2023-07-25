# VDS-OTC
V-DDS/vds-otc
pragma solidity =0.8.4;

import "@openzeppelin/contracts/utils/structs/EnumerableSet.sol";

//VDS公链OTC合约地址： ae0c807a4e89296c76a6ac298c1829bc6ab7fa11

interface ERC20 {
    function transfer(address _to, uint256 _value) external returns (bool success);

    function transferFrom(address _from,address _to,uint256 _value) external returns (bool success);

    function balanceOf(address _to) external returns (uint balances);
}

library SafeMath {
    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a + b;
        require(c >= a, "SafeMath: addition overflow");

        return c;
    }

    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b <= a, "SafeMath: subtraction overflow");
        uint256 c = a - b;
        return c;
    }

    function mul(uint256 a, uint256 b) internal pure returns (uint256) {
        if (a == 0) {
            return 0;
        }

        uint256 c = a * b;
        require(c / a == b, "SafeMath: multiplication overflow");

        return c;
    }

    function div(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b > 0, "SafeMath: division by zero");
        uint256 c = a / b;
        return c;
    }
}

contract Context {
    function _msgSender() internal view returns (address) {
        return msg.sender;
    }

    function _msgData() internal pure returns (bytes memory) {
        return msg.data;
    }
}

contract Ownable is Context {
    address private _owner;

    event OwnershipTransferred(
        address indexed previousOwner,
        address indexed newOwner
    );


    constructor() public {
        _transferOwnership(_msgSender());
    }


    function owner() public view returns (address) {
        return _owner;
    }

    modifier onlyOwner() {
        require(owner() == _msgSender(), "Ownable: caller is not the owner");
        _;
    }


    function renounceOwnership() public onlyOwner {
        _transferOwnership(address(0));
    }


    function transferOwnership(address newOwner) public onlyOwner {
        require(newOwner != address(0),"Ownable: new owner is the zero address");
        _transferOwnership(newOwner);
    }

    function _transferOwnership(address newOwner) internal {
        address oldOwner = _owner;
        _owner = newOwner;
        emit OwnershipTransferred(oldOwner, newOwner);
    }
}

contract VDDS_VDS_OTC is Ownable {
    using SafeMath for uint256;
    using EnumerableSet for EnumerableSet.AddressSet;
    using EnumerableSet for EnumerableSet.UintSet;

    constructor() public   {
        add_arbitration(0x513371E1ad11B91Dd649735B2F4818b5BF4A3367, 1);
        add_arbitration(0x513371E1ad11B91Dd649735B2F4818b5BF4A3367, 1);
        add_arbitration(0x513371E1ad11B91Dd649735B2F4818b5BF4A3367, 1);
    }

    struct info {
        uint256 number;
        uint256 coin_type;
        address from;
        address to;
        address contract_address;
        string coin;
        string to_address;
        string price;
        string remark;
    }

    struct params {
        address arbitrator;
        uint256 fee;
        uint256 status;
        uint256 create_block;
        uint256 confirm_block;
        uint256 is_handle;
        uint256 create_time;
        uint256 is_arbitration;
        uint256 _type;
    }

    uint256 public cancelBlockNumber = 10080;
    EnumerableSet.UintSet private cancelSellOrderIdList;
    EnumerableSet.UintSet private cancelBuyOrderIdList;
    uint256 public confirmBlockNumber = 10080;
    EnumerableSet.UintSet private confirmOrderIdList;
    EnumerableSet.AddressSet private arbitrationList;
    uint256 public arbitrationFeeRatio = 100;
    uint256 public orderFee = 20;
    mapping(uint256 => params) public params_info;
    mapping(uint256 => info) public order;
    uint256 public order_id = 1;
    uint256[] public orders;
    mapping(address => uint256[]) public my_order;
    uint256 public autoNumber = 20;
    mapping(address => mapping(uint256 => uint256) ) public latestPrice;
    function setBlockNumber(uint256 _cancel,uint256 _confirm) public onlyOwner{
        cancelBlockNumber = _cancel;
        confirmBlockNumber = _confirm;
    }
    function setAutoNumber(uint256 _autoNumber) public onlyOwner{
        autoNumber = _autoNumber;
    }
    function setOderFee(uint256 _fee) public {
        require(_fee < 10000);
        require(isArbitration(msg.sender) || msg.sender == owner());
        orderFee = _fee;
    }
    function setArbitrationFeeRatio(uint256 _fee) public {
        require(_fee < 10000);
        require(isArbitration(msg.sender) || msg.sender == owner());
        arbitrationFeeRatio = _fee;
    }

    function add_arbitration(address _address, uint256 _status)public onlyOwner{   
        if(1 == _status){
            arbitrationList.add(_address);
        }else{
            arbitrationList.remove(_address);
        }
    }
    function isArbitration(address _address) public view returns (bool){
        return arbitrationList.contains(_address);
    }
    function getArbitrationLength() public view returns (uint256){
        return arbitrationList.length();
    }
    function getArbitrationAt(uint256 index) public view returns (address)
    {
        return arbitrationList.at(index);
    }
    function getArbitrationList(uint256 start,uint256 end) public view returns (address[] memory) {
        if (end >= arbitrationList.length()) {
            end = arbitrationList.length() - 1;
        }
        uint256 length = end - start + 1;
        address[] memory _list = new address[](length);
        uint256 currentIndex = 0;
        for (uint256 i = start; i <= end; i++) {
            _list[currentIndex] = arbitrationList.at(i);
            currentIndex++;
        }
        return _list;
    }

    function autoConfirm() public {
        if(confirmBlockNumber > 0){
            uint256 _length = confirmOrderIdList.length();
            if(_length > 0){
                uint256 _orderId;
                uint256 _autoNumber;
                for(uint256 i = _length;i > 0; i--){
                    _orderId = confirmOrderIdList.at(i - 1);
                    if(params_info[_orderId].status == 2 && params_info[_orderId].is_handle == 1 && block.number > params_info[_orderId].create_block + confirmBlockNumber ){
                        
                        confirmOrder(_orderId,1);

                        _autoNumber++;
                        if(_autoNumber >= autoNumber){
                            break;
                        }
                    }
                }
            }
        }
    }

    function autoCancel(uint256 _type) public {
        if(cancelBlockNumber > 0 ){
            uint256 _autoNumber;
            if(0 == _type){
                uint256 _length = cancelSellOrderIdList.length();
                if(_length > 0){
                    uint256 _orderId;
                    for(uint256 i = _length;i > 0; i--){
                        _orderId = cancelSellOrderIdList.at(i - 1);
                        if(params_info[_orderId].status == 1 && block.number > params_info[_orderId].create_block + cancelBlockNumber ){
                            uint256 _number = order[_orderId].number;
                            if(params_info[_orderId].fee > 0){
                                uint256 _fee = order[_orderId].number.mul(params_info[_orderId].fee).div(10000).div(2); 
                                _number = _number.add(_fee);
                            }
                            if (order[_orderId].contract_address == address(0x0)) {
                                payable(order[_orderId].from).transfer(_number);
                            } else {
                                ERC20(order[_orderId].contract_address).transfer(order[_orderId].from,_number);
                            }

                            params_info[_orderId].status = 4;
                            cancelSellOrderIdList.remove(_orderId);
                            _autoNumber++;
                            if(_autoNumber >= autoNumber){
                                break;
                            }
                        }
                    }
                }
            }else{
                uint256 _length = cancelBuyOrderIdList.length();
                if(_length > 0){
                    uint256 _orderId;
                    for(uint256 i = _length ;i > 0; i--){
                        _orderId = cancelBuyOrderIdList.at(i - 1);
                        if(params_info[_orderId].status == 1 && block.number > params_info[_orderId].create_block + cancelBlockNumber ){
                            params_info[_orderId].status = 4;
                            cancelBuyOrderIdList.remove(_orderId);
                            _autoNumber++;
                            if(_autoNumber >= autoNumber){
                                break;
                            }
                        }
                    }
                }
            }
        }
    }

    function create_order(string memory _coin,uint256 _coin_type,uint256 _number,string memory _to_address,string memory _price,
        string memory _remark,address _contract_address,uint256 _type) public payable {
        require(_number > 0);
        require(!isArbitration(msg.sender));
        autoConfirm();
        if (_type == 0) {
            uint256 _totalNumber = _number;
            if(orderFee > 0){
                uint256 _feeNumber = _number.mul(orderFee).div(10000).div(2); 
                _totalNumber = _totalNumber.add(_feeNumber);
            }
        
            if (_contract_address == address(0x0)) {
                require(msg.value >= _totalNumber);
            } else {
                ERC20(_contract_address).transferFrom(msg.sender,address(this),_totalNumber);
            }
            order[order_id].from = msg.sender;
            autoCancel(0);
            cancelSellOrderIdList.add(order_id);
        } else {
            order[order_id].to = msg.sender;
            autoCancel(1);
            cancelBuyOrderIdList.add(order_id);
        }

        order[order_id].coin = _coin;
        order[order_id].coin_type = _coin_type;
        order[order_id].to_address = _to_address;
        order[order_id].number = _number;
        order[order_id].price = _price;
        order[order_id].remark = _remark;
        order[order_id].contract_address = _contract_address;
        params_info[order_id].create_block = block.number;
        params_info[order_id].fee = orderFee;
        params_info[order_id].status = 1;
        params_info[order_id]._type = _type;
        params_info[order_id].create_time = block.timestamp;
        orders.push(order_id);
        my_order[msg.sender].push(order_id);
        order_id = order_id + 1;
    }

    function step_forward(string memory _coin,uint256 _coin_type,uint256 _number,string memory _to_address,string memory _price,string memory _remark,
        address _contract_address,address _buy_address) public payable {
        require(_buy_address != address(0x0));
        require(_number > 0);
        require(!isArbitration(msg.sender));
        
        uint256 _totalNumber = _number;
        if(orderFee > 0){
            uint256 _feeNumber = _number.mul(orderFee).div(10000).div(2); 
            _totalNumber = _totalNumber.add(_feeNumber);
        }
        
        if (_contract_address == address(0x0)) {
            require(msg.value >= _totalNumber);
        } else {
            ERC20(_contract_address).transferFrom(msg.sender,address(this),_totalNumber);
        }
        order[order_id].from = msg.sender;
        order[order_id].to = _buy_address;
        order[order_id].coin = _coin;
        order[order_id].coin_type = _coin_type;
        order[order_id].to_address = _to_address;
        order[order_id].number = _number;
        order[order_id].price = _price;
        order[order_id].remark = _remark;
        order[order_id].contract_address = _contract_address;
        params_info[order_id].confirm_block = block.number;
        confirmOrderIdList.add(order_id);
        params_info[order_id].fee = orderFee;
        params_info[order_id].create_time = block.timestamp;
        params_info[order_id].status = 2;
        orders.push(order_id);
        my_order[msg.sender].push(order_id);
        my_order[_buy_address].push(order_id);
        order_id = order_id + 1;
    }

    function cancel(uint256 _orderid) public {
        require(1 == params_info[_orderid].status || 2 == params_info[_orderid].status);
        bool _isArbitration = false;
        if(msg.sender == order[_orderid].from || msg.sender == order[_orderid].to ){
            if (params_info[_orderid]._type == 0) {
                require( (1 == params_info[_orderid].status && msg.sender == order[_orderid].from) 
                        || (2 == params_info[_orderid].status && msg.sender == order[_orderid].to) );
            } else {
                require(msg.sender == order[_orderid].to);
            }
        }else{
            require(isArbitration(msg.sender));
            _isArbitration = true;
        }
        if ( (0 == params_info[_orderid]._type || 2 == params_info[_orderid].status) && order[_orderid].number > 0 ) {
            uint256 _number = order[_orderid].number;
           
            if(_isArbitration && arbitrationFeeRatio > 0){
                uint256 _arbitrationFee = _number.mul(arbitrationFeeRatio).div(10000);
                arbitrationBisect(_arbitrationFee,order[_orderid].contract_address);
                _number = _number.sub(_arbitrationFee);
            }

            if(params_info[_orderid].fee > 0){
                uint256 _fee = order[_orderid].number.mul(params_info[_orderid].fee).div(10000).div(2); 
                _number = _number.add(_fee);
            }            

            if (order[_orderid].contract_address == address(0x0) ) {
                payable(order[_orderid].from).transfer(_number);
            } else {
                ERC20(order[_orderid].contract_address).transfer(order[_orderid].from,_number);
            }
        }
        
        if(1 == params_info[_orderid].status){
            if (params_info[_orderid]._type == 0) {
                cancelSellOrderIdList.remove(_orderid);
            } else {
                cancelBuyOrderIdList.remove(_orderid);
            }
        }else{
            confirmOrderIdList.remove(_orderid);
        }

        params_info[_orderid].status = 4;
    }


    function apply_order(uint256 _order_id) public {
        require(msg.sender == order[_order_id].to);
        require(params_info[_order_id].is_handle == 0);
        params_info[_order_id].is_handle = 1;
        params_info[_order_id].confirm_block = block.number;
        confirmOrderIdList.add(_order_id);
    }


    function apply_arbitration(uint256 _order_id) public {
        require(msg.sender == order[_order_id].from || msg.sender == order[_order_id].to);
        params_info[_order_id].is_arbitration = 1;
        params_info[_order_id].arbitrator = msg.sender;
        confirmOrderIdList.remove(_order_id);
    }

    function getcoin_order(uint256 _limit,uint256 _pageNumber,uint256 status,address _contract_address) public view returns (uint256[] memory) {
        uint256 orderAmount = orders.length;

        if (status != 0) {
            orderAmount = getNumberOfcoinordersListings(status,_contract_address);
        }
        uint256 pageEnd = _limit * (_pageNumber + 1);
        uint256 orderSize = orderAmount >= pageEnd ? _limit : orderAmount.sub(_limit * _pageNumber);
        uint256[] memory orderids = new uint256[](orderSize);
        if (orderAmount > 0) {
            uint256 counter = 0;
            uint8 tokenIterator = 0;
            for (uint256 i = 0; i < orders.length && counter < pageEnd; i++) {
                if (status != 0) {
                    if (params_info[orders[i]].status == status && order[orders[i]].contract_address == _contract_address) {
                        if (counter >= pageEnd - _limit) {
                            orderids[tokenIterator] = orders[i];
                            tokenIterator++;
                        }
                        counter++;
                    }
                } else {
                    if (counter >= pageEnd - _limit) {
                        orderids[tokenIterator] = orders[i];
                        tokenIterator++;
                    }
                    counter++;
                }
            }
        }
        return orderids;
    }

    function getNumberOfcoinordersListings(uint256 status,address _contract_address) public view returns (uint256) {
        uint256 counter = 0;
        for (uint256 i = 0; i < orders.length; i++) {
            if (status > 0) {
                if (params_info[orders[i]].status == status &&order[orders[i]].contract_address == _contract_address) {
                    counter++;
                }
            }
        }
        return counter;
    }

    function getMyOrderNumber(address _address,address _contract_address) public view returns (uint256) {
        uint256 counter = 0;
        for (uint256 i = 0; i < my_order[_address].length; i++) {
            if ( order[my_order[_address][i]].contract_address == _contract_address){
                counter++;
            }
        }
        return counter;
    }

    function getMy_order(uint256 _limit,uint256 _pageNumber,address _address,address _contract_address) public view returns (uint256[] memory) {
        uint256 orderAmount = getMyOrderNumber(_address,_contract_address);

        uint256 pageEnd = _limit * (_pageNumber + 1);
        uint256 orderSize = orderAmount >= pageEnd? _limit : orderAmount.sub(_limit * _pageNumber);
        uint256[] memory orderids = new uint256[](orderSize);
        if (orderAmount > 0) {
            uint256 counter = 0;
            uint256 tokenIterator = 0;
            for (uint256 i = 0;i < my_order[_address].length && counter < pageEnd;i++) {
                if (order[my_order[_address][i]].contract_address == _contract_address) {
                    if (counter >= pageEnd - _limit) {
                        orderids[tokenIterator] = my_order[_address][i];
                        tokenIterator++;
                    }
                    counter++;
                }
            }
        }
        return orderids;
    }
    function pipei(uint256 _orderid) public payable {
        require(params_info[_orderid].status == 1);

        if (params_info[_orderid]._type == 1) {
            require(order[_orderid].to != msg.sender);
            uint256 _totalNumber = order[_orderid].number;
            if(orderFee > 0){
                uint256 _feeNumber = order[_orderid].number.mul(orderFee).div(10000).div(2); 
                _totalNumber = _totalNumber.add(_feeNumber);
            }
            if (order[_orderid].contract_address == address(0x0)) {
                require(msg.value >= _totalNumber);
            } else {
                ERC20(order[_orderid].contract_address).transferFrom(msg.sender,address(this),_totalNumber);
            }
            order[_orderid].from = msg.sender;
            cancelBuyOrderIdList.remove(_orderid);
        } else {
            require(order[_orderid].from != msg.sender);
            order[_orderid].to = msg.sender;
            cancelSellOrderIdList.remove(_orderid);
        }

        params_info[_orderid].status = 2;
        my_order[msg.sender].push(_orderid);
    }

    function agree(uint256 _orderid) public {
        require(params_info[_orderid].status == 2);
     
        require(msg.sender == order[_orderid].from || isArbitration(msg.sender) );
       
        confirmOrder(_orderid,0);
    }   
    function confirmOrder(uint256 _orderId,uint8 _type) private {
        uint256 _number = order[_orderId].number;
        if(_type != 1 && msg.sender != order[_orderId].from && arbitrationFeeRatio > 0){
            uint256 _arbitrationFee = order[_orderId].number.mul(arbitrationFeeRatio).div(10000);
            arbitrationBisect(_arbitrationFee,order[_orderId].contract_address);

            _number = _number.sub(_arbitrationFee);
        }
        if(params_info[_orderId].fee > 0){
            uint256 _fee = order[_orderId].number.mul(params_info[_orderId].fee).div(10000);
            arbitrationBisect(_fee,order[_orderId].contract_address);
            _number = _number.sub(_fee.div(2));
        }
        if (order[_orderId].contract_address == address(0x0)) {
            payable(order[_orderId].to).transfer(_number);
        } else {
            ERC20(order[_orderId].contract_address).transfer(order[_orderId].to,_number);
        }
        if(0 == params_info[_orderId].is_arbitration){
            confirmOrderIdList.remove(_orderId);
        }

        saveLatestPrice(_orderId);
        params_info[_orderId].status = 3;
    }

    function arbitrationBisect(uint256 _totalFee,address _tokenAddress) private {
        uint256 _number = getArbitrationLength();
        uint256 _arbitrationFee = _totalFee.div(_number);
        if(_arbitrationFee > 0){
            bool isToken = _tokenAddress != address(0x0);
            address _address;
            for(uint256 i; i < _number; i++){
                _address = getArbitrationAt(i);
                if(_address != address(0x0)){
                    if(isToken) {
                        ERC20(_tokenAddress).transfer(_address,_arbitrationFee);
                    } else {
                        payable(_address).transfer(_arbitrationFee);
                    }
                }
            }
        }
    }   

    function saveLatestPrice(uint256 _orderId) private {

        if(order[_orderId].coin_type != 0){
            latestPrice[order[_orderId].contract_address][order[_orderId].coin_type] = _orderId;
        }
    }

}
