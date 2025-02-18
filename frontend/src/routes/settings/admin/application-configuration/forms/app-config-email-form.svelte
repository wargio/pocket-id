<script lang="ts">
	import CheckboxWithLabel from '$lib/components/checkbox-with-label.svelte';
	import { openConfirmDialog } from '$lib/components/confirm-dialog';
	import FormInput from '$lib/components/form-input.svelte';
	import { Button } from '$lib/components/ui/button';
	import AppConfigService from '$lib/services/app-config-service';
	import type { AllAppConfig } from '$lib/types/application-configuration';
	import { createForm } from '$lib/utils/form-util';
	import { toast } from 'svelte-sonner';
	import { z } from 'zod';

	let {
		callback,
		appConfig
	}: {
		appConfig: AllAppConfig;
		callback: (appConfig: Partial<AllAppConfig>) => Promise<void>;
	} = $props();

	const appConfigService = new AppConfigService();

	let isSendingTestEmail = $state(false);

	const formSchema = z.object({
		smtpHost: z.string().min(1),
		smtpPort: z.number().min(1),
		smtpUser: z.string(),
		smtpPassword: z.string(),
		smtpFrom: z.string().email(),
		smtpTls: z.boolean(),
		smtpSkipCertVerify: z.boolean(),
		emailOneTimeAccessEnabled: z.boolean(),
		emailLoginNotificationEnabled: z.boolean()
	});

	const { inputs, ...form } = createForm<typeof formSchema>(formSchema, appConfig);

	async function onSubmit() {
		const data = form.validate();
		if (!data) return false;
		await callback(data);

		// Update the app config to don't display the unsaved changes warning
		Object.entries(data).forEach(([key, value]) => {
			// @ts-ignore
			appConfig[key] = value;
		});

		toast.success('Email configuration updated successfully');
		return true;
	}
	async function onTestEmail() {
		// @ts-ignore
		const hasChanges = Object.keys($inputs).some((key) => $inputs[key].value !== appConfig[key]);

		if (hasChanges) {
			openConfirmDialog({
				title: 'Save changes?',
				message:
					'You have to save the changes before sending a test email. Do you want to save now?',
				confirm: {
					label: 'Save and send',
					action: async () => {
						const saved = await onSubmit();
						if (saved) {
							sendTestEmail();
						}
					}
				}
			});
		} else {
			sendTestEmail();
		}
	}

	async function sendTestEmail() {
		isSendingTestEmail = true;
		await appConfigService
			.sendTestEmail()
			.then(() => toast.success('Test email sent successfully to your email address.'))
			.catch(() =>
				toast.error('Failed to send test email. Check the server logs for more information.')
			)
			.finally(() => (isSendingTestEmail = false));
	}
</script>

<form onsubmit={onSubmit}>
	<h4 class="text-lg font-semibold">SMTP Configuration</h4>
	<div class="mt-4 grid grid-cols-1 items-end gap-5 md:grid-cols-2">
		<FormInput label="SMTP Host" bind:input={$inputs.smtpHost} />
		<FormInput label="SMTP Port" type="number" bind:input={$inputs.smtpPort} />
		<FormInput label="SMTP User" bind:input={$inputs.smtpUser} />
		<FormInput label="SMTP Password" type="password" bind:input={$inputs.smtpPassword} />
		<FormInput label="SMTP From" bind:input={$inputs.smtpFrom} />
		<CheckboxWithLabel
			id="tls"
			label="TLS"
			description="Enable TLS for the SMTP connection."
			bind:checked={$inputs.smtpTls.value}
		/>
		<CheckboxWithLabel
			id="skip-cert-verify"
			label="Skip Certificate Verification"
			description="This can be useful for self-signed certificates."
			bind:checked={$inputs.smtpSkipCertVerify.value}
		/>
	</div>
	<h4 class="mt-10 text-lg font-semibold">Enabled Emails</h4>
	<div class="mt-4 flex flex-col gap-5">
		<CheckboxWithLabel
			id="email-login-notification"
			label="Email Login Notification"
			description="Send an email to the user when they log in from a new device."
			bind:checked={$inputs.emailLoginNotificationEnabled.value}
		/>
		<CheckboxWithLabel
			id="email-one-time-access"
			label="Email One Time Access"
			description="Allows users to sign in with a link sent to their email. This reduces the security significantly as anyone with access to the user's email can gain entry."
			bind:checked={$inputs.emailOneTimeAccessEnabled.value}
		/>
	</div>

	<div class="mt-8 flex flex-wrap justify-end gap-3">
		<Button isLoading={isSendingTestEmail} variant="secondary" onclick={onTestEmail}
			>Send test email</Button
		>
		<Button type="submit">Save</Button>
	</div>
</form>
