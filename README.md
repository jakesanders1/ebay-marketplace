    const explicitRefunds: Boolean = true;

    const buyer: Party = Role("Buyer");
    const seller: Party = Role("Seller");
    const burnAddress: Party = PK("addr1{stockpicka}");

    const price: Value = ConstantParam("Price");
    const collateral: Value = ConstantParam("Collateral amount");

    const sellerCollateralTimeout: Timeout = TimeParam("Collateral deposit by seller timeout");
    const buyerCollateralTimeout: Timeout = TimeParam("Deposit of collateral by buyer timeout");
    const depositTimeout: Timeout = TimeParam("Deposit of price by buyer timeout");
    const disputeTimeout: Timeout = TimeParam("Dispute by buyer timeout");
    const answerTimeout: Timeout = TimeParam("Complaint deadline"); -- Can be set

    function depositCollateral(party: Party, timeout: Timeout, timeoutContinuation: Contract, continuation: Contract): Contract {
        return When([Case(Deposit(party, party, ada, collateral), continuation)],
            timeout,
            timeoutContinuation);
    }

    function burnCollaterals(continuation: Contract): Contract {
        return Pay(seller, Party(burnAddress), ada, collateral,
            Pay(buyer, Party(burnAddress), ada, collateral,
                continuation));
    }

    function deposit(timeout: Timeout, timeoutContinuation: Contract, continuation: Contract): Contract {
        return When([Case(Deposit(seller, buyer, ada, price), continuation)],
            timeout,
            timeoutContinuation);
    }

    function choice(choiceName: string, chooser: Party, choiceValue: SomeNumber, continuation: Contract): Case {
        return Case(Choice(ChoiceId(choiceName, chooser),
            [Bound(choiceValue, choiceValue)]),
            continuation);
    }

    function choices(timeout: Timeout, chooser: Party, timeoutContinuation: Contract, list: { value: SomeNumber, name: string, continuation: Contract }[]): Contract {
        var caseList: Case[] = new Array(list.length);
        list.forEach((element, index) =>
            caseList[index] = choice(element.name, chooser, element.value, element.continuation)
        );
        return When(caseList, timeout, timeoutContinuation);
    }

    function sellerToBuyer(continuation: Contract): Contract {
        return Pay(seller, Account(buyer), ada, price, continuation);
    }

    function refundSellerCollateral(continuation: Contract): Contract {
        if (explicitRefunds) {
            return Pay(seller, Party(seller), ada, collateral, continuation);
        } else {
            return continuation;
        }
    }

    function refundBuyerCollateral(continuation: Contract): Contract {
        if (explicitRefunds) {
            return Pay(buyer, Party(buyer), ada, collateral, continuation);
        } else {
            return continuation;
        }
    }

    function refundCollaterals(continuation: Contract): Contract {
        return refundSellerCollateral(refundBuyerCollateral(continuation));
    }

    const refundBuyer: Contract = explicitRefunds ? Pay(buyer, Party(buyer), ada, price, Close) : Close;

    const refundSeller: Contract = explicitRefunds ? Pay(seller, Party(seller), ada, price, Close) : Close;

    const contract: Contract =
        depositCollateral(seller, sellerCollateralTimeout, Close,
            depositCollateral(buyer, buyerCollateralTimeout, refundSellerCollateral(Close),
                deposit(depositTimeout, refundCollaterals(Close),
                    choices(disputeTimeout, buyer, refundCollaterals(refundSeller),
                        [{ value: 0n, name: "I recieved my product", continuation: refundCollaterals(refundSeller) },
                        {
                            value: 1n, name: "Report an issue",
                            continuation:
                                sellerToBuyer(
                                    choices(answerTimeout, seller, refundCollaterals(refundBuyer),
                                        [{ value: 1n, name: "Confirm issue", continuation: refundCollaterals
                                        { value: 0n, name: "Dispute issue", continuation: burnCollaterals -- Need to make it send to your wallet
                        }]))));

    return contract;


})
