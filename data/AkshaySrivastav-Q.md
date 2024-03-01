1. Consider adding a public function to read the private `nextDepositId` state variable.
    ```solidity
    function getNextDepositId() external view returns (DepositIdentifier) {
        return nextDepositId;
    }
    ```