This repositry is forked to analyze and find bugs.

### 1. Incorrect use of `AccountInfo`

#### Bug

For signer `AccountInfo` is used. While `AccountInfo` is flexible, it doesn't provide any guarantees about the account's properties or permissions.

```rust
#[account(mut)]
pub signer: AccountInfo<'info>,
```

#### Fix

Added `Signer`

```rust
#[account(mut)]
pub signer: Signer<'info>,
```

### 2. Potential Integer Overflow in `transfer_points`

#### Bug

The `transfer_points` function uses `u16` for amount and points, which could lead to integer overflow if not handled carefully.

```rust
    pub fn transfer_points(ctx: Context<TransferPoints>, _id_sender:u32, _id_receiver:u32, amount: u16) -> Result<()> {

```

#### Fix

Use checked arithmetic operations

```rust
sender.points = sender.points.checked_sub(amount).ok_or(MyError::ArithmeticError)?;
receiver.points = receiver.points.checked_add(amount).ok_or(MyError::ArithmeticError)?;
```

### 3. Account Seed Issues

#### Bug

There was inconsistent use of seeds for generating Program Derived Addresses (PDAs).

```rust
#[derive(Accounts)]
pub struct CreateUser<'info> {
    #[account(
        init,
        payer = signer,
        space = 8 + 4 + 32 + (4 + 10) + 2,
        seeds = [b"user", id.to_le_bytes().as_ref()],
        bump
    )]
    pub user: Account<'info, User>,
}

#[derive(Accounts)]
pub struct TransferPoints<'info> {
    #[account(
        seeds = [b"user", id_sender.to_le_bytes().as_ref()],
        bump
    )]
    pub sender: Account<'info, User>,
    #[account(
        seeds = [b"user", id_receiver.to_le_bytes().as_ref()],
        bump
    )]
    pub receiver: Account<'info, User>,
}
```

#### Correction

Seeds are now used consistently to correctly generate PDAs, incorporating a unique identifier to ensure uniqueness of account addresses.

```rust
#[derive(Accounts)]
pub struct CreateUser<'info> {
    #[account(
        init,
        payer = signer,
        space = 8 + 4 + 32 + (4 + 10) + 2,
        seeds = [b"user", signer.key.as_ref(), id.to_le_bytes().as_ref()],
        bump
    )]
    pub user: Account<'info, User>,
}

#[derive(Accounts)]
pub struct TransferPoints<'info> {
    #[account(
        mut,
        seeds = [b"user", sender.key().as_ref()],
        bump,
        has_one = owner
    )]
    pub sender: Account<'info, User>,
    #[account(
        mut,
        seeds = [b"user", receiver.key().as_ref()],
        bump
    )]
    pub receiver: Account<'info, User>,
}
```

#### 4. Incorrect user validation

#### Bug

No verification of owner of a account in `TransferPoints`. Which allow unauthorized actions.

```rust
pub struct TransferPoints<'info> {
    #[account(
        seeds = [b"user", id_sender.to_le_bytes().as_ref()],
        bump
    )]
    pub sender: Account<'info, User>,

```

#### Fix

Added `has_one = owner`. To ensure validation.

```rust
pub struct TransferPoints<'info> {
    #[account(
        seeds = [b"user", id_sender.to_le_bytes().as_ref()],
        bump,
        has_one = owner
    )]
    pub sender: Account<'info, User>,
```
