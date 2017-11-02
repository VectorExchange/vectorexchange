# vectorexchange
Vector's Public Repository 
pragma solidity ^0.4.11;
 
library SafeMath {
    function mul(uint256 a, uint256 b) internal constant returns (uint256) {
        uint256 c = a * b;
        assert(a == 0 || c / a == b);
        return c;
    }
 
    function div(uint256 a, uint256 b) internal constant returns (uint256) {
        // assert(b > 0); // Solidity automatically throws when dividing by 0
        uint256 c = a / b;
        // assert(a == b * c + a % b); // There is no case in which this doesn't hold
        return c;
    }
 
    function sub(uint256 a, uint256 b) internal constant returns (uint256) {
        assert(b <= a);
        return a - b;
    }
 
    function add(uint256 a, uint256 b) internal constant returns (uint256) {
        uint256 c = a + b;
        assert(c >= a);
        return c;
    }
}
 
contract IERC20 {
 
    function totalSupply() public constant returns (uint256);
    function balanceOf(address who) public constant returns (uint256);
    function transfer(address to, uint256 value) public;
    function transferFrom(address from, address to, uint256 value) public;
    function approve(address spender, uint256 value) public;
    function allowance(address owner, address spender) public constant returns (uint256);
 
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
 
}
 
contract VCTRToken is IERC20 {
 
    using SafeMath for uint256;
 
    enum Stage {PREICO, BLOCKED, ICO}
 
    // Token properties
    string public name = "Vector Token";
    string public symbol = "VCTR";
    uint public decimals = 18;
 
    uint public _totalSupply = 50000000e18;
 
    uint public _icoSupply = 40000000e18; // 80%
 
    uint public _distributionSupply = 10000000e18; // 20% advisor and founder key Allocation
 
    uint public _advisorFund= 500000e18; // 1%
 
    uint public _founderAndTeamCap = 9500000e18; // 19%
 
    // Balances for each account
    mapping (address => uint256) balances;
 
    // Owner of account approves the transfer of an amount to another account
    mapping (address => mapping(address => uint256)) allowed;
 
    // start and end timestamps where investments are allowed (both inclusive)
    uint256 public startTime;
    uint256 public endTime;
 
    Stage public stage;
 
    // Owner of Token
    address public owner;
 
    // Wallet Address of Token
    address public walletAddress;
 
    // how many token units a buyer gets per wei
    uint public PRICE = 250;
 
    uint public minContribAmount = 1 ether; // 1 ether
 
    uint public softCap = 10000 ether; // 3000000 USD
 
    // amount of raised money in wei
    uint256 public fundRaised;
   
    // saving data of token
    struct VCTRTokenStruct{
        uint256 _dateReceived;
        address _addressSender;
        uint256 _amountReceived;
        bytes32 _tokenValue;
    }
    mapping(uint256 => VCTRTokenStruct) public tokensStructs;
 
    event BurnToken(uint256 value);
 
    event TokenPurchase(address indexed purchaser, address indexed beneficiary, uint256 value, uint256 amount);
 
    // modifier to allow only owner has full control on the function
    modifier onlyOwner {
        require(msg.sender == owner);
        _;
    }
 
    // Constructor
    // @notice RQXToken Contract
    // @return the transaction address
    function VCTRToken(uint256 _startTime, uint256 _endTime, address _walletAddress) public payable {
        require(_startTime >= getNow() && _endTime >= _startTime && _walletAddress != 0x0);
 
        startTime = _startTime;
        endTime = _endTime;
        walletAddress = _walletAddress;
 
        balances[walletAddress] = _totalSupply;
 
        owner = msg.sender;
 
        stage = Stage.PREICO;
    }
 
 
    // Payable method
    // @notice Anyone can buy the tokens on tokensale by paying ether
    function () public payable {
        tokensale(msg.sender);
    }
 
    // @notice tokensale
    // @param recipient The address of the recipient
    // @return the transaction address and send the event as Transfer
    function tokensale(address recipient) public payable {
        require(recipient != 0x0);
        require(validPurchase());
 
        uint discount = 2; // 50% on pre-ico first 7 days
        uint nowTime = getNow();
        uint week1 = startTime + (7 days * 1000);
        uint256 weiAmount = msg.value;
        uint tokens = weiAmount.mul(getPrice());
 
        if (stage == Stage.PREICO && nowTime <= week1) {
            tokens = tokens.mul(discount);
        }
 
        require(_icoSupply >= tokens);
        // update state
        fundRaised = fundRaised.add(weiAmount);
 
        balances[walletAddress] = balances[walletAddress].sub(tokens);
        balances[recipient] = balances[recipient].add(tokens);
 
       _icoSupply = _icoSupply.sub(tokens);
 
        TokenPurchase(msg.sender, recipient, weiAmount, tokens);
 
        forwardFunds();
    }
   
    // Withdrawal tokens to Ethers
    function safeWithdrawal(uint256 _amount) public onlyOwner{
        if (stage != Stage.ICO){
            revert();
        }
       
        if(balances[msg.sender]<_amount){
            revert();
        }
       
        if(owner == msg.sender){
            if(!owner.send(_amount)){
                revert();
            }
        }
    }
 
    // send ether to the fund collection wallet
    // override to create custom fund forwarding mechanisms
    function forwardFunds() internal {
        walletAddress.transfer(msg.value);
    }
 
    // @return true if the transaction can buy tokens
    function validPurchase() internal constant returns (bool) {
 
        uint week1 = startTime + (7 days * 1000);
        uint week4 = week1 * 4; //3 weeks of blocking. So first week for PreICO, 3 weeks,
                                //for blocking, afterwards

        //SET STAGES Blocked
        if (getNow() > week1 && getNow() < week4) {
            stage = Stage.BLOCKED;
        }

        //SET STAGES ICO
        if (getNow() > week4) {
            stage = Stage.ICO;
        }


        bool withinPeriod = false;
        if(getNow() < week1 || (getNow() > week4 && getNow() <= endTime)) {
            withinPeriod = true;
        }
        bool nonZeroPurchase = msg.value != 0;
        bool minContribution = minContribAmount <= msg.value;
        return withinPeriod && nonZeroPurchase && minContribution;
    }
 
 
    function burnToken() public onlyOwner {
         require(_icoSupply >= 0 && endTime < getNow());
         balances[walletAddress] = balances[walletAddress].sub(_icoSupply);
         _totalSupply = _totalSupply.sub(_icoSupply);
         BurnToken(_icoSupply);
         _icoSupply = 0;
    }
 
 
    // @return true if crowdsale current lot event has ended
    function hasEnded() public constant returns (bool) {
        return getNow() > endTime;
    }
 
    function getNow() public constant returns (uint) {
        return (now * 1000);
    }
 
      // Change total Supply
    function changeTotalSupply(uint _totalsupply) onlyOwner {
        _totalSupply = _totalsupply;
    }
   
    // get held token value
    function getTimeHeld(uint256 _tokenValue) public constant returns (uint256){
        return (getNow() - tokensStructs[_tokenValue]._dateReceived) / (60 * 60 * 24);
    }
   
    // get approval _date
    function getApprovalDate(uint256 _tokenValue) public constant returns (uint256){
        return tokensStructs[_tokenValue]._dateReceived;
    }
 
     // Change start time
    function changeStartDate(uint256 _starttime) onlyOwner {
         uint week1 = startTime + (7 days * 1000);
         if (stage == Stage.PREICO && getNow() <= week1) {
             startTime = _starttime;
        }
 
    }
    // @return total tokens supplied
    function totalSupply() public constant returns (uint256) {
        return _totalSupply;
    }
 
    // What is the balance of a particular account?
    // @param who The address of the particular account
    // @return the balanace the particular account
    function balanceOf(address who) public constant returns (uint256) {
        return balances[who];
    }
 
 
    // @notice send `value` token to `to` from `msg.sender`
    // @param to The address of the recipient
    // @param value The amount of token to be transferred
    // @return the transaction address and send the event as Transfer
    function transfer(address to, uint256 value) public {
        require (
            balances[msg.sender] >= value && value > 0
        );
        balances[msg.sender] = balances[msg.sender].sub(value);
        balances[to] = balances[to].add(value);
        Transfer(msg.sender, to, value);
    }
 
    // @notice send `value` token to `to` from `from`
    // @param from The address of the sender
    // @param to The address of the recipient
    // @param value The amount of token to be transferred
    // @return the transaction address and send the event as Transfer
    function transferFrom(address from, address to, uint256 value) public {
        require (
            allowed[from][msg.sender] >= value && balances[from] >= value && value > 0
        );
        balances[from] = balances[from].sub(value);
        balances[to] = balances[to].add(value);
        allowed[from][msg.sender] = allowed[from][msg.sender].sub(value);
        Transfer(from, to, value);
    }
 
    // Allow spender to withdraw from your account, multiple times, up to the value amount.
    // If this function is called again it overwrites the current allowance with value.
    // @param spender The address of the sender
    // @param value The amount to be approved
    // @return the transaction address and send the event as Approval
    function approve(address spender, uint256 value) public {
        require (
            balances[msg.sender] >= value && value > 0
        );
        allowed[msg.sender][spender] = value;
       
        // saving data
        tokensStructs[value]._amountReceived = value;
        tokensStructs[value]._addressSender = spender;
        tokensStructs[value]._dateReceived = getNow();
       
        Approval(msg.sender, spender, value);
    }
     // Token distribution to Founder & employee, Bounties, Advisors funds, Development & company budget and Longterm fund allocation
    // _founderAndTeamCap = 9500000;
    // _advisorsFundCap = 500000;
 
    function sendFounderAndTeamToken(address to, uint256 value) onlyOwner {
        require (
            to != 0x0 && value > 0 && _distributionSupply >= value
        );
 
        balances[walletAddress] = balances[walletAddress].sub(value);
        balances[to] = balances[to].add(value);
        _distributionSupply = _distributionSupply.sub(value);
        Transfer(walletAddress, to, value);
    }
 
 
    // Check the allowed value for the spender to withdraw from owner
    // @param owner The address of the owner
    // @param spender The address of the spender
    // @return the amount which spender is still allowed to withdraw from owner
    function allowance(address _owner, address spender) public constant returns (uint256) {
        return allowed[_owner][spender];
    }
 
     function getDiscount() internal constant returns (uint256) {
        uint256 discount = 0;
        uint nowTime = getNow();
 
        uint week1 = startTime + (7 days * 1000);
 
        if (stage == Stage.PREICO && nowTime <= week1) {
            discount = 50;
        }
 
        return discount;
    }
 
    // Get current price of a Token
    // @return the price or token value for a ether
    function getPrice() public constant returns (uint result) {
        return PRICE;
    }
 
    function getTokenDetail() public constant returns (string, string, uint256, uint256, uint, uint, uint) {
        return (name, symbol, startTime, endTime, _totalSupply, _icoSupply, _distributionSupply);
    }
 
    function getDistributionSupply() public constant returns (uint, uint, uint) {
        return (_distributionSupply,  _advisorFund, _founderAndTeamCap);
    }
}
