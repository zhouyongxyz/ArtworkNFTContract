# ArtWorkNFTs Contract Code Comments

## 1. ArtworkNFTs.cdc code comments

```js

// ArtworkNFTs
// NFT items for Artworks!
// ArtworkNFTs is the Artwork NFT Contarct
pub contract ArtworkNFTs: NonFungibleToken {

    // Events
    // contract was initialized
    pub event ContractInitialized()
    // when user withdraw the artwork nft
    pub event Withdraw(id: UInt64, from: Address?)
    // when user deposit the artwork nft
    pub event Deposit(id: UInt64, to: Address?)
    // when the new artwork nft be minted
    pub event Minted(id: UInt64, typeID: UInt64, hashCode: String)

    // Named Paths
    // CollectionStoragePath artwork nft storage path
    pub let CollectionStoragePath: StoragePath
    // CollectionPublicPath artwork nft public storage path
    pub let CollectionPublicPath: PublicPath
    // MinterStoragePath atwork nft minter storage path
    pub let MinterStoragePath: StoragePath

    // totalSupply
    // The total number of ArtworkNFTs that have been minted
    //
    pub var totalSupply: UInt64

    // NFT
    // A Artwork as an NFT
    //
    pub resource NFT: NonFungibleToken.INFT {
        // The token's ID
        pub let id: UInt64
        // The token's type, e.g. 0 == Image, Video
        pub let typeID: UInt64

        // the NFT hash code
        pub let hashCode: String

        // initializer
        //
        init(initID: UInt64, initTypeID: UInt64, initHashCode: String) {
            self.id = initID
            self.typeID = initTypeID
            self.hashCode = initHashCode
        }
    }

    // This is the interface that users can cast their ArtworkNFTs Collection as
    // to allow others to deposit ArtworkNFTs into their Collection. It also allows for reading
    // the details of ArtworkNFTs in the Collection.
    pub resource interface ArtworkNFTsCollectionPublic {
        pub fun deposit(token: @NonFungibleToken.NFT) // deposit the artwork nft
        pub fun getIDs(): [UInt64]  // get all the nft ids
        pub fun borrowNFT(id: UInt64): &NonFungibleToken.NFT // the borrow object of nft
        pub fun borrowArtworkNFT(id: UInt64): &ArtworkNFTs.NFT? {  // borrow object of artwork nft
            // If the result isn't nil, the id of the returned reference
            // should be the same as the argument to the function
            post {
                (result == nil) || (result?.id == id):
                    "Cannot borrow KittyItem reference: The ID of the returned reference is incorrect"
            }
        }
    }

    // Collection
    // A collection of KittyItem NFTs owned by an account
    //
    pub resource Collection: ArtworkNFTsCollectionPublic, NonFungibleToken.Provider, NonFungibleToken.Receiver, NonFungibleToken.CollectionPublic {
        // dictionary of NFT conforming tokens
        // NFT is a resource type with an `UInt64` ID field
        //
        pub var ownedNFTs: @{UInt64: NonFungibleToken.NFT}

        // withdraw
        // Removes an NFT from the collection and moves it to the caller
        //
        pub fun withdraw(withdrawID: UInt64): @NonFungibleToken.NFT {
            let token <- self.ownedNFTs.remove(key: withdrawID) ?? panic("missing NFT")

            emit Withdraw(id: token.id, from: self.owner?.address)

            return <-token
        }

        // deposit
        // Takes a NFT and adds it to the collections dictionary
        // and adds the ID to the id array
        //
        pub fun deposit(token: @NonFungibleToken.NFT) {
            let token <- token as! @ArtworkNFTs.NFT

            let id: UInt64 = token.id

            // add the new token to the dictionary which removes the old one
            let oldToken <- self.ownedNFTs[id] <- token

            emit Deposit(id: id, to: self.owner?.address)

            destroy oldToken
        }

        // getIDs
        // Returns an array of the IDs that are in the collection
        //
        pub fun getIDs(): [UInt64] {
            return self.ownedNFTs.keys
        }

        // borrowNFT
        // Gets a reference to an NFT in the collection
        // so that the caller can read its metadata and call its methods
        //
        pub fun borrowNFT(id: UInt64): &NonFungibleToken.NFT {
            return &self.ownedNFTs[id] as &NonFungibleToken.NFT
        }

        // borrowArtworkNFT
        // Gets a reference to an NFT in the collection as a KittyItem,
        // exposing all of its fields (including the typeID).
        // This is safe as there are no functions that can be called on the KittyItem.
        //
        pub fun borrowArtworkNFT(id: UInt64): &ArtworkNFTs.NFT? {
            if self.ownedNFTs[id] != nil {
                let ref = &self.ownedNFTs[id] as auth &NonFungibleToken.NFT
                return ref as! &ArtworkNFTs.NFT
            } else {
                return nil
            }
        }

        // destructor
        destroy() {
            destroy self.ownedNFTs
        }

        // initializer
        //
        init () {
            self.ownedNFTs <- {}
        }
    }

    // createEmptyCollection
    // public function that anyone can call to create a new empty collection
    //
    pub fun createEmptyCollection(): @NonFungibleToken.Collection {
        return <- create Collection()
    }

    // NFTMinter
    // Resource that an admin or something similar would own to be
    // able to mint new NFTs
    //
	pub resource NFTMinter {

		// mintNFT
        // Mints a new NFT with a new ID
		// and deposit it in the recipients collection using their collection reference
        //
		pub fun mintNFT(recipient: &{NonFungibleToken.CollectionPublic}, typeID: UInt64, hashCode: String) {
            emit Minted(id: ArtworkNFTs.totalSupply, typeID: typeID, hashCode: hashCode)

			// deposit it in the recipient's account using their reference
			recipient.deposit(token: <-create ArtworkNFTs.NFT(initID: ArtworkNFTs.totalSupply, initTypeID: typeID, initHashCode: hashCode))

            ArtworkNFTs.totalSupply = ArtworkNFTs.totalSupply + (1 as UInt64)
		}
	}

    // fetch
    // Get a reference to a KittyItem from an account's Collection, if available.
    // If an account does not have a ArtworkNFTs.Collection, panic.
    // If it has a collection but does not contain the itemID, return nil.
    // If it has a collection and that collection contains the itemID, return a reference to that.
    //
    pub fun fetch(_ from: Address, itemID: UInt64): &ArtworkNFTs.NFT? {
        let collection = getAccount(from)
            .getCapability(ArtworkNFTs.CollectionPublicPath)!
            .borrow<&ArtworkNFTs.Collection{ArtworkNFTs.ArtworkNFTsCollectionPublic}>()
            ?? panic("Couldn't get collection")
        // We trust ArtworkNFTs.Collection.borowKittyItem to get the correct itemID
        // (it checks it before returning it).
        return collection.borrowArtworkNFT(id: itemID)
    }

    // initializer
    //
	init() {
        // Set our named paths
        self.CollectionStoragePath = /storage/ArtworkNFTsCollection
        self.CollectionPublicPath = /public/ArtworkNFTsCollection
        self.MinterStoragePath = /storage/ArtworkNFTsMinter

        // Initialize the total supply
        self.totalSupply = 0

        // Create a Minter resource and save it to storage
        let minter <- create NFTMinter()
        self.account.save(<-minter, to: self.MinterStoragePath)

        emit ContractInitialized()
	}
}
```

## 2. ArtworkNFTsMarket.cdc  code comments

```js
import PigCoin from "./PigCoin.cdc"
import ArtworkNFTs from "./ArtworkNFTs.cdc"
import FungibleToken from "./FungibleToken.cdc"
import NonFungibleToken from "./NonFungibleToken.cdc"

/*
    This is a simple ArtworkNFTs initial sale contract for the DApp to use
    in order to list and sell ArtworkNFTs.

    Its structure is neither what it would be if it was the simplest possible
    market contract or if it was a complete general purpose market contract.
    Rather it's the simplest possible version of a more general purpose
    market contract that indicates how that contract might function in
    broad strokes. This has been done so that integrating with this contract
    is a useful preparatory exercise for code that will integrate with the
    later more general purpose market contract.

    It allows:
    - Anyone to create Sale Offers and place them in a collection, making it
      publicly accessible.
    - Anyone to accept the offer and buy the item.

    It notably does not handle:
    - Multiple different sale NFT contracts.
    - Multiple different payment FT contracts.
    - Splitting sale payments to multiple recipients.

 */

// ArtworkNFTsMarket is artwork nft trade market contract.
pub contract ArtworkNFTsMarket {
    // SaleOffer events.
    //
    // A sale offer has been created.
    pub event SaleOfferCreated(itemID: UInt64, price: UFix64)
    // Someone has purchased an item that was offered for sale.
    pub event SaleOfferAccepted(itemID: UInt64)
    // A sale offer has been destroyed, with or without being accepted.
    pub event SaleOfferFinished(itemID: UInt64)
    
    // Collection events.
    //
    // A sale offer has been removed from the collection of Address.
    pub event CollectionRemovedSaleOffer(itemID: UInt64, owner: Address)

    // A sale offer has been inserted into the collection of Address.
    pub event CollectionInsertedSaleOffer(
      itemID: UInt64,  // nft item unique id, Increasing from 0
      typeID: UInt64,  // artwork nft's type, 0 - image , 1 - video
      hashCode: String,  // the hashcode of the artworknft file
      owner: Address,  // owner address
      price: UFix64  // the price of the artwork nft
    )

    // Named paths
    // CollectionStoragePath artwork nft collection storage,used to 
    // save user nfts
    pub let CollectionStoragePath: StoragePath
    // CollectionPublicPath artwork nft public collection storage
    pub let CollectionPublicPath: PublicPath

    // SaleOfferPublicView
    // An interface providing a read-only view of a SaleOffer
    //
    pub resource interface SaleOfferPublicView {
        pub let itemID: UInt64 // artwork nft unique id
        pub let typeID: UInt64 // artwork nft type 0 - image, 1 - video
        pub let hashCode: String // hash code of the artwork nft file
        pub let price: UFix64 // artwork nft price
    }

    // SaleOffer
    // A ArtworkNFTs NFT being offered to sale for a set fee paid in PigCoin.
    //
    pub resource SaleOffer: SaleOfferPublicView {
        // Whether the sale has completed with someone purchasing the item.
        pub var saleCompleted: Bool

        // The ArtworkNFTs NFT ID for sale.
        pub let itemID: UInt64

        // The 'type' of NFT
        pub let typeID: UInt64

        // The 'HashCode' of NFT
        pub let hashCode: String

        // The sale payment price.
        pub let price: UFix64

        // The collection containing that ID.
        access(self) let sellerItemProvider: Capability<&ArtworkNFTs.Collection{NonFungibleToken.Provider}>

        // The PigCoin vault that will receive that payment if teh sale completes successfully.
        access(self) let sellerPaymentReceiver: Capability<&PigCoin.Vault{FungibleToken.Receiver}>

        // Called by a purchaser to accept the sale offer.
        // If they send the correct payment in PigCoin, and if the item is still available,
        // the ArtworkNFTs NFT will be placed in their ArtworkNFTs.Collection .
        //
        pub fun accept(
            buyerCollection: &ArtworkNFTs.Collection{NonFungibleToken.Receiver},
            buyerPayment: @FungibleToken.Vault
        ) {
            pre {
                buyerPayment.balance == self.price: "payment does not equal offer price"
                self.saleCompleted == false: "the sale offer has already been accepted"
            }

            self.saleCompleted = true

            self.sellerPaymentReceiver.borrow()!.deposit(from: <-buyerPayment)

            let nft <- self.sellerItemProvider.borrow()!.withdraw(withdrawID: self.itemID)
            buyerCollection.deposit(token: <-nft)

            emit SaleOfferAccepted(itemID: self.itemID)
        }

        // destructor
        //
        destroy() {
            // Whether the sale completed or not, publicize that it is being withdrawn.
            emit SaleOfferFinished(itemID: self.itemID)
        }

        // initializer
        // Take the information required to create a sale offer, notably the capability
        // to transfer the ArtworkNFTs NFT and the capability to receive PigCoin in payment.
        //
        init(
            sellerItemProvider: Capability<&ArtworkNFTs.Collection{NonFungibleToken.Provider}>,
            itemID: UInt64,
            typeID: UInt64,
            hashCode: String,
            sellerPaymentReceiver: Capability<&PigCoin.Vault{FungibleToken.Receiver}>,
            price: UFix64
        ) {
            pre {
                sellerItemProvider.borrow() != nil: "Cannot borrow seller"
                sellerPaymentReceiver.borrow() != nil: "Cannot borrow sellerPaymentReceiver"
            }

            self.saleCompleted = false

            self.sellerItemProvider = sellerItemProvider
            self.itemID = itemID

            self.sellerPaymentReceiver = sellerPaymentReceiver
            self.price = price
            self.typeID = typeID
            self.hashCode = hashCode

            emit SaleOfferCreated(itemID: self.itemID, price: self.price)
        }
    }

    // createSaleOffer
    // Make creating a SaleOffer publicly accessible.
    //
    pub fun createSaleOffer (
        sellerItemProvider: Capability<&ArtworkNFTs.Collection{NonFungibleToken.Provider}>,
        itemID: UInt64,
        typeID: UInt64,
        hashCode: String,
        sellerPaymentReceiver: Capability<&PigCoin.Vault{FungibleToken.Receiver}>,
        price: UFix64
    ): @SaleOffer {
        return <-create SaleOffer(
            sellerItemProvider: sellerItemProvider,
            itemID: itemID,
            typeID: typeID,
            hashCode: hashCode,
            sellerPaymentReceiver: sellerPaymentReceiver,
            price: price
        )
    }

    // CollectionManager
    // An interface for adding and removing SaleOffers to a collection, intended for
    // use by the collection's owner.
    //
    pub resource interface CollectionManager {
        pub fun insert(offer: @ArtworkNFTsMarket.SaleOffer)
        pub fun remove(itemID: UInt64): @SaleOffer 
    }

        // CollectionPurchaser
    // An interface to allow purchasing items via SaleOffers in a collection.
    // This function is also provided by CollectionPublic, it is here to support
    // more fine-grained access to the collection for as yet unspecified future use cases.
    //
    pub resource interface CollectionPurchaser {
        pub fun purchase(
            itemID: UInt64,
            buyerCollection: &ArtworkNFTs.Collection{NonFungibleToken.Receiver},
            buyerPayment: @FungibleToken.Vault
        )
    }

    // CollectionPublic
    // An interface to allow listing and borrowing SaleOffers, and purchasing items via SaleOffers in a collection.
    //
    pub resource interface CollectionPublic {
        pub fun getSaleOfferIDs(): [UInt64]
        pub fun borrowSaleItem(itemID: UInt64): &SaleOffer{SaleOfferPublicView}?
        pub fun purchase(
            itemID: UInt64,
            buyerCollection: &ArtworkNFTs.Collection{NonFungibleToken.Receiver},
            buyerPayment: @FungibleToken.Vault
        )
   }

    // Collection
    // A resource that allows its owner to manage a list of SaleOffers, and purchasers to interact with them.
    //
    pub resource Collection : CollectionManager, CollectionPurchaser, CollectionPublic {
        pub var saleOffers: @{UInt64: SaleOffer}

        // insert
        // Insert a SaleOffer into the collection, replacing one with the same itemID if present.
        //
         pub fun insert(offer: @ArtworkNFTsMarket.SaleOffer) {
            let itemID: UInt64 = offer.itemID
            let typeID: UInt64 = offer.typeID
            let price: UFix64 = offer.price
            let hashCode: String = offer.hashCode

            // add the new offer to the dictionary which removes the old one
            let oldOffer <- self.saleOffers[itemID] <- offer
            destroy oldOffer

            emit CollectionInsertedSaleOffer(
              itemID: itemID,
              typeID: typeID,
              hashCode: hashCode,
              owner: self.owner?.address!,
              price: price
            )
        }

        // remove
        // Remove and return a SaleOffer from the collection.
        pub fun remove(itemID: UInt64): @SaleOffer {
            emit CollectionRemovedSaleOffer(itemID: itemID, owner: self.owner?.address!)
            return <-(self.saleOffers.remove(key: itemID) ?? panic("missing SaleOffer"))
        }
 
        // purchase
        // If the caller passes a valid itemID and the item is still for sale, and passes a PigCoin vault
        // typed as a FungibleToken.Vault (PigCoin.deposit() handles the type safety of this)
        // containing the correct payment amount, this will transfer the KittyItem to the caller's
        // ArtworkNFTs collection.
        // It will then remove and destroy the offer.
        // Note that is means that events will be emitted in this order:
        //   1. Collection.CollectionRemovedSaleOffer
        //   2. ArtworkNFTs.Withdraw
        //   3. ArtworkNFTs.Deposit
        //   4. SaleOffer.SaleOfferFinished
        //
        pub fun purchase(
            itemID: UInt64,
            buyerCollection: &ArtworkNFTs.Collection{NonFungibleToken.Receiver},
            buyerPayment: @FungibleToken.Vault
        ) {
            pre {
                self.saleOffers[itemID] != nil: "SaleOffer does not exist in the collection!"
            }
            let offer <- self.remove(itemID: itemID)
            offer.accept(buyerCollection: buyerCollection, buyerPayment: <-buyerPayment)
            //FIXME: Is this correct? Or should we return it to the caller to dispose of?
            destroy offer
        }

        // getSaleOfferIDs
        // Returns an array of the IDs that are in the collection
        //
        pub fun getSaleOfferIDs(): [UInt64] {
            return self.saleOffers.keys
        }

        // borrowSaleItem
        // Returns an Optional read-only view of the SaleItem for the given itemID if it is contained by this collection.
        // The optional will be nil if the provided itemID is not present in the collection.
        //
        pub fun borrowSaleItem(itemID: UInt64): &SaleOffer{SaleOfferPublicView}? {
            if self.saleOffers[itemID] == nil {
                return nil
            } else {
                return &self.saleOffers[itemID] as &SaleOffer{SaleOfferPublicView}
            }
        }

        // destructor
        //
        destroy () {
            destroy self.saleOffers
        }

        // constructor
        //
        init () {
            self.saleOffers <- {}
        }
    }

    // createEmptyCollection
    // Make creating a Collection publicly accessible.
    //
    pub fun createEmptyCollection(): @Collection {
        return <-create Collection()
    }

    init () {
        self.CollectionStoragePath = /storage/ArtworkNFTsMarketCollection
        self.CollectionPublicPath = /public/ArtworkNFTsMarketCollection
    }
}
```



## 3. PigCoin.cdc code comments

```js
import FungibleToken from "./FungibleToken.cdc"

// PigCoin is our planform token, we use it to
// trade the artwork nfts.
pub contract PigCoin: FungibleToken {
    // TokensInitialized
    //
    // The event that is emitted when the contract is created
    pub event TokensInitialized(initialSupply: UFix64)

    // TokensWithdrawn
    //
    // The event that is emitted when tokens are withdrawn from a Vault
    pub event TokensWithdrawn(amount: UFix64, from: Address?)

    // TokensDeposited
    //
    // The event that is emitted when tokens are deposited to a Vault
    pub event TokensDeposited(amount: UFix64, to: Address?)

    // TokensMinted
    //
    // The event that is emitted when new tokens are minted
    pub event TokensMinted(amount: UFix64)

    // TokensBurned
    //
    // The event that is emitted when tokens are destroyed
    pub event TokensBurned(amount: UFix64)

    // MinterCreated
    //
    // The event that is emitted when a new minter resource is created
    pub event MinterCreated(allowedAmount: UFix64)

    // Named paths
    // VaultStoragePath vault storate path
    pub let VaultStoragePath: StoragePath
    // ReceiverPublicPath vault public path to receive token
    // from others
    pub let ReceiverPublicPath: PublicPath
    // BalancePublicPath vault public path to check the balance
    // of the vault.
    pub let BalancePublicPath: PublicPath
    // AdminStoragePath is the contract owner path, only the
    // the owner can mint the pig tokens.
    pub let AdminStoragePath: StoragePath

    // Total supply of Pigs in existence
    pub var totalSupply: UFix64

    // Vault
    //
    // Each user stores an instance of only the Vault in their storage
    // The functions in the Vault and governed by the pre and post conditions
    // in FungibleToken when they are called.
    // The checks happen at runtime whenever a function is called.
    //
    // Resources can only be created in the context of the contract that they
    // are defined in, so there is no way for a malicious user to create Vaults
    // out of thin air. A special Minter resource needs to be defined to mint
    // new tokens.
    //
    pub resource Vault: FungibleToken.Provider, FungibleToken.Receiver, FungibleToken.Balance {

        // The total balance of this vault
        pub var balance: UFix64

        // initialize the balance at resource creation time
        init(balance: UFix64) {
            self.balance = balance
        }

        // withdraw
        //
        // Function that takes an amount as an argument
        // and withdraws that amount from the Vault.
        //
        // It creates a new temporary Vault that is used to hold
        // the money that is being transferred. It returns the newly
        // created Vault to the context that called so it can be deposited
        // elsewhere.
        //
        pub fun withdraw(amount: UFix64): @FungibleToken.Vault {
            self.balance = self.balance - amount
            emit TokensWithdrawn(amount: amount, from: self.owner?.address)
            return <-create Vault(balance: amount)
        }

        // deposit
        //
        // Function that takes a Vault object as an argument and adds
        // its balance to the balance of the owners Vault.
        //
        // It is allowed to destroy the sent Vault because the Vault
        // was a temporary holder of the tokens. The Vault's balance has
        // been consumed and therefore can be destroyed.
        //
        pub fun deposit(from: @FungibleToken.Vault) {
            let vault <- from as! @PigCoin.Vault
            self.balance = self.balance + vault.balance
            emit TokensDeposited(amount: vault.balance, to: self.owner?.address)
            vault.balance = 0.0
            destroy vault
        }

        destroy() {
            PigCoin.totalSupply = PigCoin.totalSupply - self.balance
            if(self.balance > 0.0) {
                emit TokensBurned(amount: self.balance)
            }
        }
    }

    // createEmptyVault
    //
    // Function that creates a new Vault with a balance of zero
    // and returns it to the calling context. A user must call this function
    // and store the returned Vault in their storage in order to allow their
    // account to be able to receive deposits of this token type.
    //
    pub fun createEmptyVault(): @Vault {
        return <-create Vault(balance: 0.0)
    }

    pub resource Administrator {

        // createNewMinter
        //
        // Function that creates and returns a new minter resource
        //
        pub fun createNewMinter(allowedAmount: UFix64): @Minter {
            emit MinterCreated(allowedAmount: allowedAmount)
            return <-create Minter(allowedAmount: allowedAmount)
        }
    }

    // Minter
    //
    // Resource object that token admin accounts can hold to mint new tokens.
    //
    pub resource Minter {

        // The amount of tokens that the minter is allowed to mint
        pub var allowedAmount: UFix64

        // mintTokens
        //
        // Function that mints new tokens, adds them to the total supply,
        // and returns them to the calling context.
        //
        pub fun mintTokens(amount: UFix64): @PigCoin.Vault {
            pre {
                amount > 0.0: "Amount minted must be greater than zero"
                amount <= self.allowedAmount: "Amount minted must be less than the allowed amount"
            }
            PigCoin.totalSupply = PigCoin.totalSupply + amount
            self.allowedAmount = self.allowedAmount - amount
            emit TokensMinted(amount: amount)
            return <-create Vault(balance: amount)
        }

        init(allowedAmount: UFix64) {
            self.allowedAmount = allowedAmount
        }
    }

    init() {
        // Set our named paths.
        self.VaultStoragePath = /storage/PigCoinVault
        self.ReceiverPublicPath = /public/PigCoinReceiver
        self.BalancePublicPath = /public/PigCoinBalance
        self.AdminStoragePath = /storage/PigCoinAdmin

        // Initialize contract state.
        self.totalSupply = 0.0

        // Create the one true Admin object and deposit it into the conttract account.
        let admin <- create Administrator()
        self.account.save(<-admin, to: self.AdminStoragePath)

        // Emit an event that shows that the contract was initialized.
        emit TokensInitialized(initialSupply: self.totalSupply)
    }
}

```

