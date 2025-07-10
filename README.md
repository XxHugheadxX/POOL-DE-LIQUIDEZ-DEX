# POOL-DE-LIQUIDEZ-DEX
ETHKIPU
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract TokenA {
    string public name = "Token A";
    string public symbol = "TKA";
    uint8 public decimals = 18;
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    constructor() {
        totalSupply = 1_000_000 * 10 ** decimals; // 1 millón de tokens
        balanceOf[msg.sender] = totalSupply;
        emit Transfer(address(0), msg.sender, totalSupply);
    }

    function transfer(address to, uint256 value) public returns (bool) {
        require(to != address(0), "Invalid address");
        require(balanceOf[msg.sender] >= value, "Insufficient balance");
        balanceOf[msg.sender] -= value;
        balanceOf[to] += value;
        emit Transfer(msg.sender, to, value);
        return true;
    }

    function approve(address spender, uint256 value) public returns (bool) {
        allowance[msg.sender][spender] = value;
        emit Approval(msg.sender, spender, value);
        return true;
    }

    function transferFrom(address from, address to, uint256 value) public returns (bool) {
        require(to != address(0), "Invalid address");
        require(balanceOf[from] >= value, "Insufficient balance");
        require(allowance[from][msg.sender] >= value, "Insufficient allowance");
        balanceOf[from] -= value;
        balanceOf[to] += value;
        allowance[from][msg.sender] -= value;
        emit Transfer(from, to, value);
        return true;
    }
}
TOKEN B
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract TokenB {
    string public name = "Token B";
    string public symbol = "TKB";
    uint8 public decimals = 18;
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    constructor() {
        totalSupply = 1_000_000 * 10 ** decimals; // 1 millón de tokens
        balanceOf[msg.sender] = totalSupply;
        emit Transfer(address(0), msg.sender, totalSupply);
    }

    function transfer(address to, uint256 value) public returns (bool) {
        require(to != address(0), "Invalid address");
        require(balanceOf[msg.sender] >= value, "Insufficient balance");
        balanceOf[msg.sender] -= value;
        balanceOf[to] += value;
        emit Transfer(msg.sender, to, value);
        return true;
    }

    function approve(address spender, uint256 value) public returns (bool) {
        allowance[msg.sender][spender] = value;
        emit Approval(msg.sender, spender, value);
        return true;
    }

    function transferFrom(address from, address to, uint256 value) public returns (bool) {
        require(to != address(0), "Invalid address");
        require(balanceOf[from] >= value, "Insufficient balance");
        require(allowance[from][msg.sender] >= value, "Insufficient allowance");
        balanceOf[from] -= value;
        balanceOf[to] += value;
        allowance[from][msg.sender] -= value;
        emit Transfer(from, to, value);
        return true;
    }
}


DEX


// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IERC20 {
    function transfer(address to, uint256 value) external returns (bool);
    function transferFrom(address from, address to, uint256 value) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

contract SimpleDEX {
    address public owner;
    IERC20 public tokenA;
    IERC20 public tokenB;
    uint256 public reserveA; // Reserva de TokenA en el pool
    uint256 public reserveB; // Reserva de TokenB en el pool
    uint256 public constant K = 10**36; // Constante para el producto constante (ajustada para 18 decimales)

    event LiquidityAdded(address indexed provider, uint256 amountA, uint256 amountB);
    event LiquidityRemoved(address indexed provider, uint256 amountA, uint256 amountB);
    event Swap(address indexed user, address indexed tokenIn, uint256 amountIn, uint256 amountOut);

    constructor(address _tokenA, address _tokenB) {
        owner = msg.sender;
        tokenA = IERC20(_tokenA);
        tokenB = IERC20(_tokenB);
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call this function");
        _;
    }

    // Añadir liquidez al pool
    function addLiquidity(uint256 amountA, uint256 amountB) external onlyOwner {
        require(amountA > 0 && amountB > 0, "Amounts must be greater than 0");
        
        // Transferir tokens al contrato
        require(tokenA.transferFrom(msg.sender, address(this), amountA), "TokenA transfer failed");
        require(tokenB.transferFrom(msg.sender, address(this), amountB), "TokenB transfer failed");

        // Actualizar reservas
        reserveA += amountA;
        reserveB += amountB;

        emit LiquidityAdded(msg.sender, amountA, amountB);
    }

    // Retirar liquidez del pool
    function removeLiquidity(uint256 amountA, uint256 amountB) external onlyOwner {
        require(amountA <= reserveA && amountB <= reserveB, "Insufficient reserves");
        require(amountA > 0 && amountB > 0, "Amounts must be greater than 0");

        // Transferir tokens al propietario
        require(tokenA.transfer(msg.sender, amountA), "TokenA transfer failed");
        require(tokenB.transfer(msg.sender, amountB), "TokenB transfer failed");

        // Actualizar reservas
        reserveA -= amountA;
        reserveB -= amountB;

        emit LiquidityRemoved(msg.sender, amountA, amountB);
    }

    // Intercambiar TokenA por TokenB
    function swapAforB(uint256 amountAIn) external {
        require(amountAIn > 0, "Amount must be greater than 0");
        require(reserveA > 0 && reserveB > 0, "Pool is empty");

        // Calcular cantidad de TokenB a recibir usando la fórmula del producto constante
        uint256 amountBOut = (reserveB * amountAIn) / (reserveA + amountAIn);

        // Verificar que la cantidad a enviar sea mayor que 0
        require(amountBOut > 0 && amountBOut < reserveB, "Invalid output amount");

        // Transferir TokenA al contrato
        require(tokenA.transferFrom(msg.sender, address(this), amountAIn), "TokenA transfer failed");
        // Transferir TokenB al usuario
        require(tokenB.transfer(msg.sender, amountBOut), "TokenB transfer failed");

        // Actualizar reservas
        reserveA += amountAIn;
        reserveB -= amountBOut;

        emit Swap(msg.sender, address(tokenA), amountAIn, amountBOut);
    }

    // Intercambiar TokenB por TokenA
    function swapBforA(uint256 amountBIn) external {
        require(amountBIn > 0, "Amount must be greater than 0");
        require(reserveA > 0 && reserveB > 0, "Pool is empty");

        // Calcular cantidad de TokenA a recibir usando la fórmula del producto constante
        uint256 amountAOut = (reserveA * amountBIn) / (reserveB + amountBIn);

        // Verificar que la cantidad a enviar sea mayor que 0
        require(amountAOut > 0 && amountAOut < reserveA, "Invalid output amount");

        // Transferir TokenB al contrato
        require(tokenB.transferFrom(msg.sender, address(this), amountBIn), "TokenB transfer failed");
        // Transferir TokenA al usuario
        require(tokenA.transfer(msg.sender, amountAOut), "TokenA transfer failed");

        // Actualizar reservas
        reserveB += amountBIn;
        reserveA -= amountAOut;

        emit Swap(msg.sender, address(tokenB), amountBIn, amountAOut);
    }

    // Obtener el precio de un token en términos del otro
    function getPrice(address _token) external view returns (uint256) {
        require(_token == address(tokenA) || _token == address(tokenB), "Invalid token");
        require(reserveA > 0 && reserveB > 0, "Pool is empty");

        if (_token == address(tokenA)) {
            // Precio de TokenA en términos de TokenB (cuántos TokenB por 1 TokenA)
            return (reserveB * 10**18) / reserveA;
        } else {
            // Precio de TokenB en términos de TokenA (cuántos TokenA por 1 TokenB)
            return (reserveA * 10**18) / reserveB;
        }
    }
}
