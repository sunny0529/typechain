## KMS signTx 개발 고려사항

### 1. keystore - sign transaction 추가

* 1. src/blockchains/[체인이름]/keyStore.ts 에서, signTx() 에 코드 작성하기

    ```jsx
    /*
      static signTx(node: BIP32Interface, rawTx: RawTx): SignedTx {
        // ...
      }
    */
    ```

    - transaction instruction type에 따라 1. transaction을 생성하여 2. transaction에 sign한 후 3. signedTx를 반환하는 함수를 구현하면 된다.
    - transaction instruction type을 인자로 넣는 이유는 어떤 종류의 transaction이 들어오든 signTx()는 각 상황에 맞게 transaction에 sign하는 기능을 할 수 있도록 확장 가능해야 하기 때문이다.
    - input으로 들어오는 rawTx의 대략적인 형태

        ```jsx
        const TRANSFER = 0;
        const DELEGATE = 1;

        rarTx: RawTx = {
              recentBlockHash, // (recentBlockHash는 예시) transaction instruction 종류에 상관없이 공통적으로 사용되는 값들 넣기
              ixs: [ // instruction을 배열로 입력하는 형식
                {
                  amount: "1", // 각 transaction type 마다 필요한 값 입력
                  transactionType: TRANSFER, // transaction type
                },
                {
                  amount: "1", // 각 transaction type 마다 필요한 값 입력
                  gas: "3000000", // 각 transaction type 마다 필요한 값 입력
                  transactionType: DELEGATE, // transaction type
                },
                //... add new transaction instructions
              ],
            }
        ```

    1. transaction 생성
        - 한번에 여러 종류의 instruction들을 모아서 하나의 transaction으로 만들 수 있는 경우

            ```jsx
            // 각 instruction type에 맞게 instruction 생성해서 반환하는 함수
            function createInstruction(ix) { 
              switch (ix.transactionType) {
                case 0: {
                  // transfer
                  return transactions.transfer(ix.amount);
                }
                case 1: {
                  // delegate
                  return transactions.functionCall(
            				ix.gas,
                    ix.amount
                  );
                }
                default:
                  break;
              }
            throw new Error("Create instrauction error");
            }

            // rawTx 입력으로 받은 여러 개의 instruction을 하나의 transaction으로 만들어 반환하는 함수 
            function createTransaction(rawTx) { // 
            	try {
            	    const transaction = new Transaction({
            	      recentBlockhash: rawTx.recentBlockhash,
            	    });
            	    for (let i = 0; i < rawTx.ixs.length; i += 1) {
            	      transaction.add(createInstruction(rawTx.ixs[i]));
            	    }
            	    return transaction;
            	  } catch (error) {
            	    throw new Error(error);
            	  }
            }
            ```

        - 만약 keystore와 ledger에서 createTransaction 코드가 동일하다면 하나로 빼서 사용하기

            src/blockchains/[체인이름]/createTransaction.ts 로 따로 빼기

    2. transaction에 sign 코드 작성
    3. signedTx를 반환
        - input으로 받았던 rawTx과, sign된 transaction을 반환하면 된다.

            ```jsx
            SignedTx {
              rawTx: RawTx;
              signedTx?: any;
            }
            ```

#### 2. src/keyStore.ts 에서, signTxFromKeyStore() 에 add blockchain

    ```jsx
    export async function signTxFromKeyStore(
      path: BIP44,
      keyStore: KeyStore,
      password: string,
      rawTx: RawTx
    ): Promise<{ [key: string]: any } | null> {
      try {
        const key = await getAlgo2HashKey(password, keyStore);
        if (key && keyStore) {
          const mnemonic = await JWE.createDecrypt(key).decrypt(
            keyStore.j.join(".")
          );
          const seed = mnemonicToSeedSync(mnemonic.plaintext.toString());
          const node = fromSeed(seed);
          const child = node.derivePath(
            `m/44'/${path.type}'/${path.account}'/0/${path.index}`
          );

          switch (path.type) {
            // blockchains
            case CHAIN.MINA: {
              const response = mina.signTx(child, rawTx);
              return { ...response };
            }
            case CHAIN.NEAR: {
              const response = near.signTx(seed, path, rawTx);
              return { ...response };
            }
            case CHAIN.SOLANA: {
              const response = solana.signTx(seed, path, rawTx);
              return { ...response };
            }
            // add blockchains....
            // blockchains
            default:
              break;
          }
        }
        return null;
      } catch (error) {
        // eslint-disable-next-line no-console
        console.error(error);
        return null;
      }
    }
    ```

#### 3. test/keystore/[체인이름].ts 에서, test file 만들어서 test 해보기
    - kms에서 구현한 signTx() rawTx값과 함께 호출하고
    - verifySignature할 수 있으면 true, false 반환하기

### Ⅱ. ledger - sign transaction 추가

#### 1. src/blockchains/[체인이름]/ledger.ts 에서, signTx() 에 코드 작성하기

    ```jsx
    /*
      static async signTx(
        path: BIP44,
        transport: Transport,
        rawTx: RawTx
      ): Promise<SignedTx> {
        // ...
      }
    */
    ```

    - 만약 keystore에서 만든 createTransaction 코드를 사용할 수 있다면,

        src/blockchains/[체인이름]/createTransaction.ts 에  하나로 빼서 사용하기 

    - 아래 부터는 keystore의 설명과 동일
    - transaction instruction type에 따라 1. transaction을 생성하여 2. transaction에 sign한 후 3. signedTx를 반환하는 함수를 구현하면 된다.
    - transaction instruction type을 인자로 넣는 이유는 어떤 종류의 transaction이 들어오든 signTx()는 각 상황에 맞게 transaction에 sign하는 기능을 할 수 있도록 확장 가능해야 하기 때문이다.
    - input으로 들어오는 rawTx의 대략적인 형태

        ```jsx
        const TRANSFER = 0;
        const DELEGATE = 1;

        rarTx: RawTx = {
              recentBlockHash, // (recentBlockHash는 예시) transaction instruction 종류에 상관없이 공통적으로 사용되는 값들 넣기
              ixs: [ // instruction을 배열로 입력하는 형식
                {
                  amount: "1", // 각 transaction type 마다 필요한 값 입력
                  transactionType: TRANSFER, // transaction type
                },
                {
                  amount: "1", // 각 transaction type 마다 필요한 값 입력
                  gas: "3000000", // 각 transaction type 마다 필요한 값 입력
                  transactionType: DELEGATE, // transaction type
                },
                //... add new transaction instructions
              ],
            }
        ```

    1. transaction 생성
        - 한번에 여러 종류의 instruction들을 모아서 하나의 transaction으로 만들 수 있는 경우

            ```jsx
            // 각 instruction type에 맞게 instruction 생성해서 반환하는 함수
            function createInstruction(ix) { 
              switch (ix.transactionType) {
                case 0: {
                  // transfer
                  return transactions.transfer(ix.amount);
                }
                case 1: {
                  // delegate
                  return transactions.functionCall(
            				ix.gas,
                    ix.amount
                  );
                }
                default:
                  break;
              }
            throw new Error("Create instrauction error");
            }

            // rawTx 입력으로 받은 여러 개의 instruction을 하나의 transaction으로 만들어 반환하는 함수 
            function createTransaction(rawTx) { // 
            	try {
            	    const transaction = new Transaction({
            	      recentBlockhash: rawTx.recentBlockhash,
            	    });
            	    for (let i = 0; i < rawTx.ixs.length; i += 1) {
            	      transaction.add(createInstruction(rawTx.ixs[i]));
            	    }
            	    return transaction;
            	  } catch (error) {
            	    throw new Error(error);
            	  }
            }
            ```

        - 만약 keystore와 ledger에서 createTransaction 코드가 동일하다면 하나로 빼서 사용하기

            src/blockchains/[체인이름]/createTransaction.ts 로 따로 빼기

    2. transaction에 sign 코드 작성
    3. signedTx를 반환
        - input으로 받았던 rawTx과, sign된 transaction을 반환하면 된다.

            ```jsx
            SignedTx {
              rawTx: RawTx;
              signedTx?: any;
            }
            ```

#### 2. src/ledger.ts 에서, signTxFromLedger() 에 add blockchain

    ```jsx
    export async function signTxFromLedger(
      path: BIP44,
      transport: Transport,
      rawTx: RawTx
    ): Promise<{ [key: string]: any } | null> {
      try {
        const key = await getAlgo2HashKey(password, keyStore);
        if (key && keyStore) {
          const mnemonic = await JWE.createDecrypt(key).decrypt(
            keyStore.j.join(".")
          );
          const seed = mnemonicToSeedSync(mnemonic.plaintext.toString());
          const node = fromSeed(seed);
          const child = node.derivePath(
            `m/44'/${path.type}'/${path.account}'/0/${path.index}`
          );

          switch (path.type) {
            // blockchains
            case CHAIN.MINA: {
              const response = mina.signTx(child, rawTx);
              return { ...response };
            }
            case CHAIN.NEAR: {
              const response = near.signTx(seed, path, rawTx);
              return { ...response };
            }
            case CHAIN.SOLANA: {
              const response = solana.signTx(seed, path, rawTx);
              return { ...response };
            }
            // add blockchains....
            // blockchains
            default:
              break;
          }
        }
        return null;
      } catch (error) {
        // eslint-disable-next-line no-console
        console.error(error);
        return null;
      }
    }
    ```

#### 3. test/ledger/[체인이름].ts 에서, test file 만들어서 test 해보기
    - kms에서 구현한 signTx() rawTx값과 함께 호출하고
    - verifySignature할 수 있으면 true, false 반환하기

### III. 주의 사항

#### 1. keystore의 signTx()와 ledger의 signTx()가 같은 rawTx를 input으로 받아서 같은 output을 return하도록 구현해야 함
#### 2. trnasaction type에 추가해야하는 것들 - delegate와 undelegate
#### 3. 체인마다 예외가 되는 사항들 정리하기 (front에서 hint로 주석 달아야 함)
#### 4. error 처리
    - 최종적으로 transaction send는 안되는데, sign까지 잘되거나 verifySignature까지도 true로 잘 되는 경우들을 에러 처리해야 한다.
    - [있을 수 있는 대표적인 상황들] 이 중에서 보라색 하이라이트는 부분은 꼭 확인하기

        **Ⅰ. RawTx 넘길 때 몇몇 필드가 빠진 상황**

        1. instruction이 없을 때

            - TransactionType 필드없이 넘길 때

            - 다음과 같이 처리

                ```jsx
                function createInstruction(ix: any): Transaction | TransactionInstruction {
                  if (typeof ix.transactionType !== "number") {
                    throw new Error("Instruction has no transaction type");
                    // TransactionType 필드없이 넘길 때 처리
                  }
                  switch (ix.transactionType) {
                    case 0: {
                			//...
                    }
                    default:
                      break;
                  }
                  throw new Error("Create instrauction error"); 
                  //null로 return하지 않도록 처리 필요
                }

                function createTransaction(rawTx) {
                	try {
                	    const transaction = new Transaction({
                	      recentBlockhash: rawTx.recentBlockhash,
                	    });
                	    for (let i = 0; i < rawTx.ixs.length; i += 1) {
                	      transaction.add(createInstruction(rawTx.ixs[i]));
                	    }
                	    return transaction;
                	  } catch (error) {
                	    throw new Error(error);
                	  }
                }
                ```

            - Isx없이 넘길 때

            - 다음과 같이 처리

                ```jsx
                static async signTx(
                    seed: Buffer,
                    path: BIP44,
                    rawTx: RawTx
                  ): Promise<SignedTx> {
                    const payer = KEYSTORE.getKeypair(seed, path);
                    const transaction = createTransaction(rawTx);
                    if (transaction.instructions.length === 0) {
                      throw new Error("No instructions provided");
                    } 
                // rawTx로 받은 inxs가 0개이면 다음과 같이 처리해서 사인 못 하도록 처리 필요
                    transaction.sign(<Signer>payer);
                    return {
                      rawTx,
                      signedTx: transaction,
                    };
                  }
                ```

        2. 다른 field들도 없이 보내면 어떻게 되는지 test해보기

        **Ⅱ. 각 종 유효하지 않은 값 넣을때**

        1. amount에 다른 타입넣으면 어떻게 되는지 test해보기
            - 다만, string은 숫자로 변환해서 가능하게끔 처리해야 한다.
            - 다음과 같이 처리

                ```jsx
                //number 대신 SDK에서 지원하는 type으로 지정하기
                // ex) numder or hex
                if (
                        typeof ix.amount !== "number" && 
                        typeof ix.amount !== "string"
                      ) {
                        throw new Error("Amount is required number");
                      }
                      const lamports = Number(ix.amount);
                //string to number(or hex)
                ```

        2. 다른 field들도 다른 type넣으면 어떻게 되는지 test해보기
