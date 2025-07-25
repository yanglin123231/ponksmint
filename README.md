# ponksmint
ponksnft
// Anchor Program: Mint NFT with BONK token payment (integrated with Metaplex)

use anchor_lang::prelude::*;
use anchor_spl::token::{self, Mint, Token, TokenAccount, Transfer};
use mpl_token_metadata::instruction::{create_metadata_accounts_v3, CreateMetadataAccountsV3Args, DataV2};
use anchor_lang::solana_program::{program::invoke_signed, system_program, sysvar};

const BONK_MINT: &str = "GJubF7LkDsF1URXN8dN5Fs8HfxGp1ryX4J5pGAvPbonk"; // Replace with real BONK mint address
const BONK_AMOUNT: u64 = 1_000_000; // 1 BONK (adjust based on BONK decimals)

#[program]
pub mod bonk_nft_mint {
    use super::*;

    pub fn mint_nft_with_bonk(
        ctx: Context<MintWithBonk>,
        name: String,
        symbol: String,
        uri: String,
    ) -> Result<()> {
        // 1. Transfer BONK to treasury
        let cpi_ctx = CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            Transfer {
                from: ctx.accounts.user_bonk_account.to_account_info(),
                to: ctx.accounts.treasury_bonk_account.to_account_info(),
                authority: ctx.accounts.user.to_account_info(),
            },
        );
        token::transfer(cpi_ctx, BONK_AMOUNT)?;

        // 2. Create Metadata Account
        let metadata_accounts = vec![
            ctx.accounts.metadata.to_account_info(),
            ctx.accounts.nft_mint.to_account_info(),
            ctx.accounts.nft_mint_authority.to_account_info(),
            ctx.accounts.user.to_account_info(),
            ctx.accounts.token_metadata_program.to_account_info(),
            ctx.accounts.rent.to_account_info(),
            ctx.accounts.system_program.to_account_info(),
        ];

        let data = DataV2 {
            name,
            symbol,
            uri,
            seller_fee_basis_points: 0,
            creators: None,
            collection: None,
            uses: None,
        };

        invoke_signed(
            &create_metadata_accounts_v3(
                ctx.accounts.token_metadata_program.key(),
                ctx.accounts.metadata.key(),
                ctx.accounts.nft_mint.key(),
                ctx.accounts.nft_mint_authority.key(),
                ctx.accounts.user.key(),
                ctx.accounts.user.key(),
                data,
                true,
                true,
                None,
                None,
                None,
            ),
            &metadata_accounts,
            &[&[b"mint-authority", &[ctx.bumps["nft_mint_authority"]]]],
        )?;

        Ok(())
    }
}

#[derive(Accounts)]
pub struct MintWithBonk<'info> {
    #[account(mut)]
    pub user: Signer<'info>,

    #[account(
        mut,
        constraint = user_bonk_account.mint == bonk_mint.key()
    )]
    pub user_bonk_account: Account<'info, TokenAccount>,

    #[account(mut)]
    pub treasury_bonk_account: Account<'info, TokenAccount>,

    #[account(
        address = bonk_mint.key()
    )]
    pub bonk_mint: Account<'info, Mint>,

    #[account(
        init,
        payer = user,
        mint::decimals = 0,
        mint::authority = nft_mint_authority,
        mint::freeze_authority = nft_mint_authority
    )]
    pub nft_mint: Account<'info, Mint>,

    /// CHECK: This is a PDA used to mint NFTs
    #[account(
        seeds = [b"mint-authority"],
        bump
    )]
    pub nft_mint_authority: UncheckedAccount<'info>,

    /// CHECK: Metadata account PDA from Metaplex
    #[account(mut)]
    pub metadata: UncheckedAccount<'info>,

    /// CHECK: Metaplex Token Metadata program
    pub token_metadata_program: UncheckedAccount<'info>,

    pub rent: Sysvar<'info, Rent>,
    pub system_program: Program<'info, System>,
    pub token_program: Program<'info, Token>,
}
