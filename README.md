// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// Define the ResearchToken contract
contract ResearchToken {
    string public name = "ResearchToken";
    string public symbol = "RCH";
    uint8 public decimals = 18;
    uint256 public totalSupply;
    address public owner;

    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    constructor() {
        owner = msg.sender;
        totalSupply = 1000000 * 10 ** uint256(decimals); // 1 million tokens
        balanceOf[owner] = totalSupply;
        emit Transfer(address(0), owner, totalSupply);
    }

    function transfer(address _to, uint256 _value) public returns (bool success) {
        require(balanceOf[msg.sender] >= _value, "Insufficient balance");
        require(_to != address(0), "Invalid address");

        balanceOf[msg.sender] -= _value;
        balanceOf[_to] += _value;
        emit Transfer(msg.sender, _to, _value);
        return true;
    }

    function approve(address _spender, uint256 _value) public returns (bool success) {
        allowance[msg.sender][_spender] = _value;
        emit Approval(msg.sender, _spender, _value);
        return true;
    }

    function transferFrom(address _from, address _to, uint256 _value) public returns (bool success) {
        require(balanceOf[_from] >= _value, "Insufficient balance");
        require(allowance[_from][msg.sender] >= _value, "Allowance exceeded");
        require(_to != address(0), "Invalid address");

        balanceOf[_from] -= _value;
        balanceOf[_to] += _value;
        allowance[_from][msg.sender] -= _value;
        emit Transfer(_from, _to, _value);
        return true;
    }
}

// Define the ResearchProject contract
contract ResearchProject {
    enum ProjectStatus { Active, Completed, Failed }

    struct Project {
        string name;
        address researcher;
        uint256 fundingGoal;
        uint256 fundsRaised;
        ProjectStatus status;
    }

    Project[] public projects;
    ResearchToken public token;
    address public owner;

    event ProjectCreated(uint256 projectId, string name, address researcher, uint256 fundingGoal);
    event Funded(uint256 projectId, address funder, uint256 amount);
    event RewardClaimed(uint256 projectId, address researcher, uint256 reward);

    modifier onlyOwner() {
        require(msg.sender == owner, "Not the contract owner");
        _;
    }

    constructor(address tokenAddress) {
        token = ResearchToken(tokenAddress);
        owner = msg.sender;
    }

    // Function to create a new research project
    function createProject(string memory _name, uint256 _fundingGoal) external onlyOwner {
        projects.push(Project({
            name: _name,
            researcher: msg.sender,
            fundingGoal: _fundingGoal,
            fundsRaised: 0,
            status: ProjectStatus.Active
        }));
        
        emit ProjectCreated(projects.length - 1, _name, msg.sender, _fundingGoal);
    }

    // Function to fund a project
    function fundProject(uint256 _projectId, uint256 _amount) external {
        require(_projectId < projects.length, "Invalid project ID");
        require(projects[_projectId].status == ProjectStatus.Active, "Project is not active");

        // Transfer tokens from funder to this contract
        require(token.transferFrom(msg.sender, address(this), _amount), "Transfer failed");

        // Update project funds
        projects[_projectId].fundsRaised += _amount;

        emit Funded(_projectId, msg.sender, _amount);
    }

    // Function for researcher to claim rewards when funding goal is reached
    function claimReward(uint256 _projectId) external {
        require(_projectId < projects.length, "Invalid project ID");
        Project storage project = projects[_projectId];
        require(msg.sender == project.researcher, "Only the researcher can claim rewards");
        require(project.fundsRaised >= project.fundingGoal, "Funding goal not reached");
        require(project.status == ProjectStatus.Active, "Project is not active");

        // Deactivate the project
        project.status = ProjectStatus.Completed;

        // Transfer the funds to the researcher
        require(token.transfer(msg.sender, project.fundsRaised), "Transfer failed");

        emit RewardClaimed(_projectId, msg.sender, project.fundsRaised);
    }
}
